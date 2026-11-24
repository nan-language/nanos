# 0152. DMA confined by the IOMMU

Status: accepted

## Context

Chapter 15 confines each device's DMA to an IOMMU domain of its group, and chapter 09 makes a device capability so confined no trust escape. NaN enables the IOMMU through `boot::Kernel.iommu`, only once, and returns a linear `hw::Iommu` that nothing releases; `Iommu.domain` makes an I/O domain held by the domain run that holds it, whose settlement closes it before freeing the DMA buffers made in its space. `PciFunction.attach` confines only an isolated function, one whose DMA reaches the IOMMU under its own requester id and no peer otherwise; any other function attaches only to physical memory, and one aliased behind a bridge to conventional PCI not even there. Attaching enables bus mastering. Physical memory as a DMA space is a device operation; DMA buffers, register windows, interrupts and I/O domains are confined operations. NaN reports no DMA fault (nan-language/nan#288) and has no domain shared by a group (nan-language/nan#290). On the q35 machine the SATA controller, function 2 of the multi-function device `00:1f` without ACS, is not isolated.

## Decision

- **Start step 8.** The core calls `k.iommu()` once the clock runs. The first task holds the `hw::Iommu` for the boot and lends it to the keeper. A DMAR listing a unit the call does not enable gives `fault.iommu-failed` with the unit's register base and the first condition the DMAR fails: several units, reserved memory regions, no interrupt remapping, or else the unit itself.
- **The keeper attaches.** A claim of a PCI function attaches it before its driver's Env takes it: to a new I/O domain, or to physical memory without an IOMMU or when the IOMMU refuses the function as not isolated. A claim for which no domain or table memory is left fails. The keeper remembers where each claimed function's DMA reaches, and `devices` reads it as the record's `dma` once the driver runs.
- **Drivers are confined code.** A driver receives the function, its domain and the space its buffers are made in, and ends the domain after releasing the function. The `ahci`, `nvme` and `virtio` modules allow `confined` rather than `device`; the keeper alone names physical memory.
- **Groups.** Each listed PCI function carries its IOMMU group on a machine with `iommu`: the requester id its DMA reaches the IOMMU under, its own or that of the outermost bridge to conventional PCI above it.

Alternatives rejected:

- **Each driver attaching its function:** every driver would name physical memory, staying a trust escape, and `devices` would learn confinement from each driver.
- **Refusing a function the IOMMU cannot isolate:** the sandbox's SATA disk would have no driver; a base driver, vendor code, may run unconfined as on a machine without `iommu`.

## Consequences

- `/live/security` reports `dma: confined` and each confined device `dma: confined`; the sandbox's SATA controller stays `unconfined` until NaN attaches groups (N60).
- A driver programming its device outside its domain changes nothing, and no event tells of it (N58).
- Bus mastering starts at the claim, before the driver resets its device; the empty domain absorbs a stray DMA.
