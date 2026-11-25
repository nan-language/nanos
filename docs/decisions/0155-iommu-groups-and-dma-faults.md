# 0155. IOMMU groups and DMA faults

Status: accepted

## Context

Chapter 15 confines each device's DMA to the IOMMU domain of its group, binds a group's functions only to drivers of one principal and provenance class, and answers a DMA fault with `device.dma-fault` and the abandonment of the driver's domain; chapter 09 makes a device capability so confined no trust escape. NaN enables the IOMMU through `boot::Kernel.iommu`, once, and returns a linear `hw::Iommu`. `Iommu.domain` makes an I/O domain, a resource of the domain run holding it. `PciFunction.group` names a function's group, and `attach` keeps the attached functions of a group in one space. A DMA buffer is unmapped by `into_box` or by the settlement of the domain run holding it. `Iommu.fault` returns each recorded fault with the bus path of its DMA requester id, and `Iommu.fault_irq` installs the interrupt announcing them. Every driver acts for `system`. Physical memory as a DMA space is a device operation; DMA buffers, register windows, interrupts and I/O domains are confined operations.

## Decision

- **Start step 8.** The core calls `k.iommu()` once the clock runs and hands the `hw::Iommu` to the keeper. A DMAR listing a unit the call does not enable gives `fault.iommu-failed` with the unit's register base and the first condition the DMAR fails: several units, reserved memory regions, no interrupt remapping, or else the unit itself.
- **One domain per group.** A claim of a PCI function attaches it, before its driver's Env takes it, to the I/O domain of its group, which the keeper makes for the first function of the group it attaches and holds for the boot; to physical memory without an IOMMU or when the IOMMU does not translate the function. A claim for which no domain or table memory is left fails. The keeper remembers where each claimed function's DMA reaches, and `devices` reads it as the record's `dma` once the driver runs.
- **Drivers are confined code.** A driver receives the function and the space its buffers are made in, and releases the function. The `ahci`, `nvme` and `virtio` modules allow `confined` rather than `device`; the keeper alone names physical memory.
- **Group conflicts.** Each listed PCI function carries its group on a machine with `iommu`, and each `drivers` entry the provenance class of its code, `vendor` for the kernel image's. `devices` binds no device to an entry of another class than a driver bound in its group: the device is `failed(device.group-conflict)`.
- **DMA faults.** The keeper installs the IOMMU's fault interrupt on the signal of its hot-plug slots. For each fault it writes `device.dma-fault` with the bus path, the page's address and the access, and abandons the domain of each driver whose function has that bus path or whose group it names, through the abandoner each claim carries.

Alternatives rejected:

- **A new domain per claim, closed by its driver:** the functions of a group bound to several drivers share one space, and settlement already unmaps each driver's buffers.
- **Each driver attaching its function:** every driver would name physical memory, staying a trust escape, and `devices` would learn confinement from each driver.

## Consequences

- `/live/security` reports `dma: confined`, and every device the IOMMU translates `dma: confined`, the sandbox's SATA controller in the group of `00:1f` included.
- The functions of a group reach each other's buffers while they are mapped; their drivers share a provenance class.
- A fault under a group's DMA requester id abandons the drivers of every function of that group.
- Bus mastering starts at the claim, before the driver resets its device; a stray DMA faults in the group's domain.
