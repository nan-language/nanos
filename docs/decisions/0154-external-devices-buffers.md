# 0154. External devices' buffers

Status: accepted

## Context

Chapter 15 maps the buffers of internal devices once, when allocated, and those of external devices, below a port the firmware marks external-facing or hot-pluggable, at each submission, unmapping them at completion. NaN maps a buffer into its I/O domain when `hw::DmaShared.of` makes it, and `into_box` unmaps it, returning once the IOMMU no longer translates it. The base drivers copy each request's data through buffers they allocate when they bind. A PCI slot tells whether it takes hot-plugged devices; a root port's `_DSD` marks it external-facing, and only the AML evaluator, an instance `devices` creates, reads it, while the keeper in the core attaches each function for its driver.

## Decision

- **External functions.** The keeper attaches a function as external when it sits below a port whose slot takes hot-plugged devices, or below a root port of a PCI root bridge whose `_DSD` sets `ExternalFacingPort` under the UUID of those properties. Discovery has the evaluator read those ports and tells the keeper before any driver binds.
- **Buffers.** `drivers::dma::Buffers` holds the buffers a driver hands its device with each request: mapped once, when made, for an internal function; for an external one, mapped at each submission and unmapped at its completion, or once the device is reset. Queues, rings and the areas a device reads between requests stay mapped. A driver copies a read's data out before the unmapping.

Alternatives rejected:

- **Unmapping before copying a read's data out:** each driver would read its buffers through a second path, for data the device controls anyway and that arrives `Untrusted`.

## Consequences

- Each request to an external device maps its buffers and waits for an IOTLB invalidation per buffer at its completion.
- A port marked external-facing below another bridge than a root bridge counts only when its slot takes hot-plugged devices.
