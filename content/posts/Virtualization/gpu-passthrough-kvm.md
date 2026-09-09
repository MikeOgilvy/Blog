+++
date = '2026-09-09T20:00:00+08:00'
draft = false
title = 'Making RTX 4060 Passthrough Work on an AMD Laptop'
tags = ["Linux", "KVM", "VFIO"]
+++

GPU passthrough is usually presented as a short checklist: enable IOMMU, bind the GPU to `vfio-pci`, add a PCI host device, and boot the guest. That model breaks down on laptops, where firmware controls GPU enumeration and also supplies the ACPI tables that describe IOMMU isolation.

On my ASUS ROG Zephyrus G14 GA403UV, the RTX 4060 was in a clean IOMMU group and was correctly bound to VFIO, yet QEMU still refused to start the VM. The useful part of the exercise was not the final XML. It was isolating the failure to an incorrect IVRS table and fixing only the affected PCI bus.

## The Topology

The host runs Fedora 43 on a Ryzen 9 8945HS. The Radeon 780M remains the host display adapter; the mobile RTX 4060 is reserved for a Windows 11 guest named `win11`.

The NVIDIA card exposes two PCI functions:

```text
0000:01:00.0  NVIDIA RTX 4060 Mobile       10de:28e0
0000:01:00.1  NVIDIA HDMI/DP audio          10de:22be
```

Both functions are the only members of IOMMU group 14. That is the topology required for a conventional VFIO assignment: the GPU and its audio function can be isolated together, while the host keeps using the 780M.

The first prerequisite is making the dGPU visible. In the laptop's Integrated mode the 4060 is removed from the PCI bus entirely, so there is nothing for VFIO to bind. Switching to Hybrid mode and rebooting makes `01:00.*` enumerable without taking the host's display away from the 780M.

## The Failure Signal

After adding both PCI functions to the Q35/UEFI `win11` domain, startup failed with:

```text
Firmware has requested this device have a 1:1 IOMMU mapping
Failed to set group container: Invalid argument
```

This message is easy to misdiagnose as a libvirt or driver problem. The important observations were:

1. `lspci -nnk` showed both functions owned by `vfio-pci`.
2. Group 14 contained no unrelated devices.
3. The same failure occurred with both the legacy VFIO container backend and QEMU's `iommufd` path.

That moved the investigation below libvirt. The kernel was rejecting the IOMMU domain before QEMU could map guest memory.

## Reading the IVRS Table

AMD systems describe IOMMU device ranges through the ACPI IVRS table. The firmware on this machine contained two type `0x22` ranges covering:

```text
0x0000..0x0fff
```

Those ranges accidentally included PCI bus `01`, where the RTX 4060 lives. Linux therefore treated the card as requiring an immutable 1:1 mapping. VFIO correctly refused to create a DMA container that violated that request.

Changing kernel parameters did not address the actual defect. `iommu=pt` changes the default mapping policy, but it does not rewrite a device-specific IVRS reservation. Likewise, `force_isolation` and no-IOMMU modes would either leave the reservation intact or weaken the security boundary.

## A Surgical ACPI Override

The fix was an ACPI table override that removes only bus `01` from those two ranges. Each range is split into:

```text
0x0000..0x00ff
0x0200..0x0fff
```

The gap corresponds to the affected bus. Every other device keeps the firmware's original isolation rules.

The patcher is deliberately conservative: it checks the IVRS signature, expected record layout, table length, and checksum before writing an output file. The resulting AML is installed through dracut's early CPIO mechanism:

```text
/etc/vfio-acpi-override/IVRS.aml
/etc/dracut.conf.d/91-vfio-ivrs.conf
```

After reboot, the kernel reported both the initramfs table and the table upgrade:

```text
ACPI: IVRS ACPI table found in initrd
ACPI: Table Upgrade: override [IVRS- AMD  -AmdTable]
```

The active table was 536 bytes, matching the patched artifact. More importantly, group 14 no longer contained the blocking `direct` reservation. It retained only the normal MSI and reserved regions.

This is the decisive test: the override changed the kernel's IOMMU model, not merely a userspace configuration file.

## Binding Before Userspace

Binding the card interactively after the desktop has started is fragile. On this laptop the GPU could be detached, but detaching the HDMI audio function could block while `snd_hda_intel` and the graphics session still held references.

The reliable approach is early binding from the initramfs:

```text
vfio-pci.ids=10de:28e0,10de:22be
```

The initramfs also preloads:

```text
vfio
vfio_pci
vfio_iommu_type1
```

The post-reboot invariant is simple and scriptable:

```text
01:00.0 ... Kernel driver in use: vfio-pci
01:00.1 ... Kernel driver in use: vfio-pci
```

I kept a separate static VFIO boot entry containing the patched IVRS table and forced device binding. That entry is a recovery path if a normal graphics-mode change ever produces an unusable host.

## The Guest Configuration

The `win11` domain uses Q35, UEFI, and a host-passthrough CPU. Both PCI functions are declared as managed host devices. A SPICE/QXL display remains enabled as a rescue console while the NVIDIA driver is installed.

The guest-side checks are different from the host-side checks. First verify that QEMU exposes the devices on the guest PCI bus; then verify Windows has loaded the NVIDIA driver. A virt-manager window showing SPICE does not mean the guest is rendering on the virtual adapter. In Windows, assigning an application to the high-performance GPU and checking its GPU Engine (`GPU 1 - 3D` or `GPU 1 - Compute`) provides a useful runtime signal.

## The Final Display and Input Path

I did not use Looking Glass in the final setup. The G14's right-side USB-C port is wired directly to the RTX 4060, so the simplest and most predictable solution was to connect that port to an external monitor with a USB-C-to-DisplayPort cable.

The resulting path is direct:

```text
Windows application -> RTX 4060 rendering -> USB-C/DisplayPort -> external monitor
```

This avoids an additional capture and shared-memory layer, and it makes the display driven by the same physical GPU that runs the Windows applications. The laptop's internal panel remains attached to the Radeon 780M and continues to display Fedora.

I also passed through the external keyboard and mouse as individual USB devices rather than passing through the entire USB controller. That keeps the laptop keyboard, touchpad, and host USB devices available to Fedora while the guest is running. The USB devices use an optional startup policy, so `win11` can still boot when they are unplugged.

In this setup the guest receives the two USB devices below:

```text
Keyboard  258a:010c
Mouse     35bb:d3fc
```

In virt-manager these are added as USB Host Devices with `startupPolicy="optional"`. Passing through the whole controller would provide lower-level access, but it would also take the laptop's built-in peripherals and every device on that controller away from the host.

SPICE/QXL remains configured as a recovery console. It is useful for installation and troubleshooting, but it is not the normal display path once the external monitor is connected to the dGPU.

## Switching Modes Safely

Once static passthrough was stable, `supergfxctl` was configured with three explicit modes:

```bash
gfx-i       # Integrated: host uses the Radeon 780M
gfx-h       # Hybrid: host may use the NVIDIA driver
gfx-v       # Vfio: reserve the 4060 for win11
gfx-status  # show mode and PCI driver ownership
```

The `Vfio -> Integrated -> Vfio` path was tested successfully, including a subsequent `win11` boot. The constraints are important:

- `win11` must be fully powered off before handing the card back to VFIO;
- both `01:00.0` and `01:00.1` must switch together;
- an external display driven by the NVIDIA stack must be disconnected or disabled before switching to VFIO;
- Hybrid transitions commonly require a GNOME logout;
- the static VFIO boot entry remains the fallback.

The audio function has no independent reset path, so ad-hoc hot-unbind is not a good replacement for the tested `supergfxctl` sequence.

## What I Verify

Every change is checked at four layers:

1. PCI ownership: both functions use `vfio-pci` when the guest is stopped.
2. IOMMU isolation: group 14 contains only the two 4060 functions and no forbidden `direct` region.
3. QEMU/libvirt state: `win11` is running and QEMU owns both host devices.
4. Kernel health: no new `1:1 IOMMU mapping`, VFIO fault, AER, or NVIDIA Xid messages.

The general lesson is that GPU passthrough failures are not always virtualization failures. If the group is clean and the devices are already bound to VFIO, inspect the firmware's IOMMU description before changing libvirt XML or weakening isolation. In this case, a narrowly-scoped IVRS correction was enough to turn a reproducible `Invalid argument` into a working, testable passthrough setup.
