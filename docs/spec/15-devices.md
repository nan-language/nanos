# Devices

This chapter defines device discovery, the device manager with its records and operations, drivers with their matching and device capabilities, volumes and device roles, the device protocols `Block`, `Console`, `Entropy`, `Rtc`, `Nic` and `Tpm`, DMA and IOMMU domains, interrupts and polling, device power, reset and cut-off, the device classes the base supports, the `fat` component, and the `device` code family.

## Device discovery

The **device manager** is the [base instance](20-base.md#base-instances) `devices`. It holds the **device root**: the capability of every device of the machine, minted at its start from the firmware tables, the PCIe configuration space and the vectors and IOMMU domains the core allocates ([vectors](04-kernel.md#vectors)). The core gives `devices` the PCIe configuration windows ([start sequence](04-kernel.md#start-sequence)). `devices` reads three sources through validators ([NaN 15 §Validators](https://github.com/nan-language/nan/blob/main/docs/spec/15-untrusted.md#validators)), ignoring a table that fails and emitting `device.table-invalid`.

1. ACPI: SPCR (console UART), TPM2 (TPM interface), and the DSDT and SSDTs (the AML namespace).
2. The device tree, on AArch64 machines booted without ACPI: nodes with `compatible`, `reg`, `interrupts` and `iommus`, and `/chosen` with `stdout-path`.
3. PCIe, through the ECAM windows of MCFG or of `pci-host-ecam-generic` nodes. `devices` scans every bus below each host bridge, depth first, reading each function's identifiers, BARs and capabilities, and keeps the bus numbers and BAR addresses the firmware assigned; a function with an unassigned memory BAR gets status `failed(device.bar-unassigned)`. A machine without ECAM has no PCI devices.

The AML evaluator is the instance `devices.aml`, which holds the AML table bytes and no device capability. It evaluates only `\_S5`, `_PRT` and `_CRS`, and reads `_HID`, `_CID`, `_UID` and `_ADR` when they are data objects. An evaluation that reaches an operation region, runs more than 100,000 AML opcodes or allocates more than 1 MiB fails, and the device it describes gets status `failed(device.aml)`.

With native PCIe hot plug, a presence change on a downstream port makes `devices` rescan below it: a new function is added and matched; a removed function is cut off ([reset](#reset)) and its record deleted.

## The device manager

Each discovered device is a `LiveDevice` record at `/live/devices/<bus path>` ([`/live`](13-tree.md#live)). A **bus path** names a device by where it sits: a PCI function by segment, bus, device and function in lower-case hexadecimal, an ACPI device by `_HID` and `_UID`, a device-tree node by its path.

```ebnf
BUS_PATH = PCI_PATH | ACPI_PATH | DT_PATH ;
PCI_PATH = "pci/" HEX HEX HEX HEX "." HEX HEX "." HEX HEX "." HEX ;
ACPI_PATH = "acpi/" ( "A" .. "Z" | "0" .. "9" )+ "." ( "0" .. "9" )+ ;
DT_PATH = "dt" ( "/" ( "a" .. "z" | "0" .. "9" | "," | "." | "_" | "+" | "-" | "@" )+ )+ ;
HEX = "0" .. "9" | "a" .. "f" ;
```

The sandbox's AHCI controller is `pci/0000.00.1f.2`; a 16550 whose `_UID` is 1 is `acpi/PNP0501.1`.

| Field | Type | Meaning |
| --- | --- | --- |
| `ids` | `DeviceIds` | PCI vendor, device, subsystem vendor, subsystem, class code and revision; ACPI `_HID`, `_CID`s and `_UID`; or device-tree `compatible` strings |
| `resources` | list of `Resource` | MMIO and port windows (base, size), interrupt sources (`msix`, `msi`, `intx`, `line`) with counts |
| `iommu_group` | `Option[u32]` | on a machine with `iommu` |
| `dma` | `none`, `confined` or `unconfined` | the DMA confinement of its driver |
| `entry` | `Option[text]` | the `drivers` entry of its driver |
| `driver` | `Option[Moniker]` | its driver instance |
| `role` | `Option[Role]` | [volumes and roles](#volumes-and-roles) |
| `power` | `on`, `suspended` or `off` | [device power](#device-power) |
| `status` | `ReconcileStatus` | [desired and observed state](13-tree.md#desired-and-observed-state), its code from this chapter or [placement](09-authority.md#placement) |
| `counters` | `DeviceCounters` | interrupts, requests, bytes read and written, errors, resets, DMA faults |

A device that no driver matches is `converged` with no driver.

Desired state is the configuration record of `devices`, `/sys/config/devices`: `DevicesConfig { bindings: map of BusPath to Binding }`. A `Binding` holds `driver`, a `DriverChoice`: `Automatic` (the default), `Unbound` or `Entry(name)` naming a `drivers` entry; and `power`, a `PowerPolicy`: `Auto` (the default), `On` or `Off`.

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `bind` | `device: BusPath`, `driver: text` | — | `device.unknown`, `device.driver-mismatch` | `recorded` |
| `unbind` | `device: BusPath` | — | `device.unknown` | `recorded` |
| `power` | `device: BusPath`, `policy: PowerPolicy` | — | `device.unknown` | `recorded` |
| `reset` | `device: BusPath` | — | `device.unknown`, `device.no-driver` | `restorable` |

`bind`, `unbind` and `power` write the device's binding (`unbind` sets `Unbound`), and `devices` reconciles the device to it.

## Drivers

### Matching

The `drivers` registry is a load-time registry ([registries](10-code.md#registries)): its entries come from the kernel image and from loaded bundles (requires [N6](22-nan.md#n6)).

| Field | Type | Meaning |
| --- | --- | --- |
| `name` | text, the key | 1 to 20 bytes of `a`–`z`, digits and `-`, starting and ending with a letter |
| `matches` | list of `DeviceMatch` | the identifiers it drives |
| `protocol` | `HashValue` | the hash of the device protocol its instances serve |
| `component` | text | the canonical path of the component state type its instances run |
| `priority` | `u8` | the tie-break among equally specific matches, higher first |

A `DeviceMatch` is one of, from most to least specific: `PciSubsystem { vendor, device, subsystem_vendor, subsystem }`; `PciDevice { vendor, device }`; `PciClass { class, mask }` on the 24-bit class code; `Compatible(text)`, more specific at an earlier position in the node's `compatible` list; `AcpiId(text)`, a `_HID` above a `_CID`; `AcpiTable(signature)`, the device a table describes (`TPM2`).

`devices` chooses a device's driver by these steps:

1. A binding `Entry(name)` uses that entry when one of its matches matches the device, else the device gets `failed(device.driver-mismatch)`; a binding `Unbound` leaves it unbound.
2. Otherwise the most specific matching entry of the kernel image or of a bundle of provenance `vendor` or `reviewed` ([provenance](09-authority.md#provenance)) wins; ties go to the higher `priority`, then to an entry of the kernel image, then to the lower `name`. An entry of a bundle of provenance `agent` drives a device only through a binding `Entry(name)`.
3. With no such entry the device stays unbound.

Loading or unloading a bundle that contributes `drivers` entries reruns these steps for every unbound device and every binding naming a contributed entry; a device bound under step 2 keeps its driver. The [preview](14-changes.md#preview) of an `install` step lists each binding it causes as an activation.

### Driver instances

A **driver** is an [instance](06-execution.md#instances) of the component its entry names, which `devices` starts for one device. Its moniker is `devices.<name><n>`, with n the lowest index no other instance of that entry uses (`devices.ahci0`). Its [spec](06-execution.md#instance-specifications) is `volatile`, `permanent` and of class `normal`, and its `grants` name a `Device` grant on its bus path ([objects and rights](09-authority.md#objects-and-rights)), from which its Env receives the **device capability** of its device, an OS [capability kind](09-authority.md#capability-kinds) (requires [N27](22-nan.md#n27)), of one of two kinds.

| `PciFunction` part | Rule |
| --- | --- |
| configuration | reads anywhere in the function's configuration space; writes anywhere except the protected registers: the command register's I/O space, memory space and bus master bits, the BARs, the expansion ROM BAR, the MSI and MSI-X capabilities, the power management control register, the ATS and ACS capabilities, and a bridge's bus number and window registers. A protected write fails with `device.config-protected`. |
| BAR windows | an uncached `hw::Mmio` window per memory BAR and an `hw::Ports` window per I/O BAR, as large as the BAR decodes, without the pages of the MSI-X table and pending bit array |
| vectors | the vectors `devices` allocated ([interrupts and polling](#interrupts-and-polling)) |
| DMA allocator | buffers for the function ([DMA and IOMMU](#dma-and-iommu)) |
| reset | a function level reset when the function supports one, else `device.unsupported` |

A `PlatformDevice` holds the windows of its `_CRS` or `reg`, the interrupt lines of its `_CRS` or `interrupts`, and a DMA allocator when the firmware describes it as a DMA master.

The windows, interrupt bindings (`hw::IrqShared`) and DMA rings (`hw::DmaShared`) are entries of the driver's resource table ([runner tables](06-execution.md#runner-tables)) (requires [N28](22-nan.md#n28)). A window needs no trusted premise: the device capability vouches that it decodes only its device (requires [N13](22-nan.md#n13)).

`devices` binds a driver by these steps:

1. It allocates the device's vectors and, on a machine with `iommu`, a new IOMMU domain for its group, and enables memory and I/O decoding; with no vector left the device gets `failed(device.no-vectors)`.
2. It writes the driver's spec at `/sys/instances/<moniker>` and its `Device` grant, held by `inst.<moniker>`, in one commit. Before the store is mounted, it starts the driver from these records and commits them once the store is mounted.
3. The runner [starts](06-execution.md#starting) the driver, and `devices` enables bus mastering once the Env holds the DMA allocator. A [placement](09-authority.md#placement) refusal gives the device `failed` with the placement code.
4. It has the runner call the driver's `health()` ([system interfaces](06-execution.md#system-interfaces)). No return within 5 s gives `failed(device.start-timeout)`, a trap or `Unhealthy` gives `failed(device.unhealthy)`, and either cuts the device off.
5. It emits `device.bound` and resolves the device's role.

Unbinding removes the spec and the grant in one commit. At boot, `devices` binds again the drivers whose specs name a device it finds, and removes the other driver specs.

## Volumes and roles

A `Block` driver serves every **unit** of its controller (a disk or a namespace), each a record at `/live/devices/<bus path>/u<n>` holding its `BlockInfo`. `devices` reads each unit's partition table through the driver with a validator: a GPT (the primary header at block 1 and its entries, checked by their CRC32s, else the backup at the last block), else an MBR's four primary entries, else, when block 0 is a FAT boot sector, one partition covering the unit. Each partition is a record at `/live/devices/<bus path>/u<n>/p<m>`, m counting from 1 in table order, with `type` (GPT type GUID or MBR type byte), `unique` (the GPT unique GUID, when present), `first` and `blocks` (in blocks), and `name` (`Untrusted` text).

A **volume view** is an instance serving `Block` with one unit, 0, over a range of a unit: `devices.<driver>-u<n>` covers unit n and `devices.<driver>-u<n>p<m>` its partition m. It checks every request against its range (`device.out-of-range`) and forwards it to the driver. `devices` starts a view when a role or a wiring needs it; a driver's own `Block` port reaches only `devices` and views.

A **role** names a device a base instance needs. A [wiring](09-authority.md#env-construction) entry may name a role in place of a moniker, resolving to the role's driver or volume view. A wiring naming a role no device holds fails with `device.role-unresolved`.

| Role | Holder |
| --- | --- |
| `console` | the UART that SPCR or `stdout-path` names, else the 16550 at port `0x3F8`, else the first UART in bus path order |
| `boot-volume` | the partition the handoff record names (by GPT unique GUID, or MBR disk signature and partition number), else the first ESP, else the first FAT partition, in bus path order |
| `store-volume` | the `nanos store` partition that [placement](12-store-format.md#placement) chooses, else the view `devices.store-img` over the boot volume's file `/nanos/store.img` |
| `entropy`, `nic` | every `Entropy` device, every `Nic` device |
| `rtc`, `tpm` | the first such device in bus path order |

A device whose driver's bundle has provenance `agent` ([provenance](09-authority.md#provenance)) holds no role, and neither do its units and partitions; each holder the table names is chosen among the other devices, units and partitions.

For `devices.store-img`, once `fat` runs ([start order](20-base.md#start-order)), `devices` has `fat` create the file as [placement](12-store-format.md#placement) sizes it when it is absent (`allocate`), then takes its block range (`extent`); a file whose clusters are not contiguous leaves `store-volume` unresolved (`device.not-contiguous`).

## Device protocols

A **device protocol** is a [protocol](07-communication.md#protocols) that drivers serve, each operation with its [declared class](14-changes.md#declared-classes) (requires [N4](22-nan.md#n4)). Every operation is a request; one waiting for a condition keeps its reply pending until the condition holds ([replies](07-communication.md#replies)). Results carrying device data are `Untrusted` ([NaN 23 §DMA](https://github.com/nan-language/nan/blob/main/docs/spec/23-low-level.md#dma)), also across bundles and spaces (requires [N29](22-nan.md#n29)). Each protocol states its driver rules: queues, interrupts or polling, timeouts and runtime suspend.

### Block

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `info` | `unit: u16` | `BlockInfo` | `device.out-of-range` | `none` |
| `read` | `unit: u16`, `block: u64`, `count: u32` | `Untrusted[Bytes]` of `count` blocks | `device.out-of-range`, `device.too-large`, `device.io`, `device.timeout`, `device.reset` | `none` |
| `write` | `unit: u16`, `block: u64`, `data: Bytes`, `fua: bool` | — | those of `read`, `device.unaligned`, `device.read-only` | `irreversible{erase}` |
| `flush` | `unit: u16` | — | `device.io`, `device.timeout`, `device.reset` | `none` |
| `discard` | `unit: u16`, `block: u64`, `count: u32` | — | `device.out-of-range`, `device.unsupported`, `device.io`, `device.reset` | `irreversible{erase}` |

`BlockInfo` holds `units: u16`, `block_bytes: u32` (512 or 4,096), `blocks: u64`, `max_transfer_bytes: u32` (a multiple of `block_bytes`, at least 64 KiB), `zone_blocks: u32` (0 on a unit that is not zoned), and the flags `cache`, `fua`, `discard` and `read_only`.

- A transfer above `max_transfer_bytes` fails with `device.too-large`, and `data` of partial blocks with `device.unaligned`.
- Requests complete in any order. A `write` is durable when it completes with `fua`, or once a `flush` sent after its completion completes; without a volatile cache, every write is durable at completion.
- On a zoned unit a write starts at its zone's write pointer (`device.unaligned`), and a `discard` of exactly one zone resets it.
- Driver rules: one queue pair per CPU when the device offers that many, else shared round-robin; one MSI-X vector per queue, bound to its CPU; completions by interrupt with device coalescing of at most 100 µs, never polled. A request not completed in 30 s is aborted and fails with `device.timeout`; a failed abort resets the controller, failing its other requests with `device.reset`. No runtime suspend; under `PowerPolicy.Auto` the driver enables the device's link power management.

### Console

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `write` | `data: Bytes` (at most 4 KiB) | — | `device.too-large`, `device.reset` | `irreversible{emit}` |
| `read` | `max: u32` | `Untrusted[Bytes]` of 1 to `max` bytes | `device.reset` | `none` |
| `write_reserved` | `data: Bytes` (at most 4 KiB) | — | `device.too-large`, `device.reset` | `irreversible{emit}` |
| `read_reserved` | `max: u32` | `Untrusted[Bytes]` of 1 to `max` bytes | `device.reset` | `none` |

- The `_reserved` operations form the **reserved lane** of the `console` [base instance](20-base.md#base-instances); operation masks keep every other holder off it. Reserved output goes ahead of queued output, at the end of the current line or after 256 bytes without a line feed. While a `read_reserved` is pending, input goes to it, never to `read`.
- `write` replies once its bytes are queued, and waits while 64 KiB are queued.
- Driver rules: receive and transmit by interrupt, the receive FIFO interrupting at 8 bytes and on timeout; never polled or suspended. Line settings stay as the firmware left them unless the driver's configuration record (`/sys/config/<moniker>`) sets `baud`, `data_bits`, `parity` and `stop_bits`.

### Entropy

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `fill` | `count: u32` (1 to 4,096) | `Untrusted[Bytes]` of `count` bytes | `device.io`, `device.reset` | `none` |

- The driver runs the repetition count and adaptive proportion tests of NIST SP 800-90B §4.4 on the raw output. A failed test makes `fill` fail with `device.io` and `health()` return `Unhealthy` with reason `entropy`, until the driver restarts.
- `devices` reseeds the system random pool from the `entropy` role ([entropy](03-boot.md#entropy)).
- Driver rules: an interrupt per completed request; runtime suspend after 10 s without a request.

### Rtc

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `read` | — | `utc_s: i64` | `device.io` | `none` |
| `set` | `utc_s: i64` | — | `device.io` | `restorable` |

- The RTC holds UTC in seconds. The [clock](08-scheduling.md#clocks) instance is its only client.
- Driver rules: no interrupts; never suspended.

### Nic

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `info` | — | `NicInfo` | — | `none` |
| `link` | `known: LinkState` | the `LinkState` once it differs from `known` | `device.reset` | `none` |
| `send` | `queue: u16`, `frames: list of Bytes` | — | `device.too-large`, `device.link-down`, `device.reset` | `irreversible{emit}` |
| `receive` | `queue: u16`, `max: u32` | list of 1 to `max` `Untrusted[Bytes]` frames | `device.reset` | `none` |
| `filter` | `multicast: list of Bytes`, `promiscuous: bool` | — | `device.unsupported`, `device.reset` | `restorable` |

- `NicInfo` holds `medium` (`ethernet` or `vsock`), `address` (the 6-byte MAC, or the 8-byte little-endian guest context id), `mtu_bytes: u32` (of frame payload), `queues: u16`, and the offload flags `rx_checksum`, `tx_checksum` and `tso`. `LinkState` holds `up: bool` and `full_duplex: bool`.
- An `ethernet` frame starts at its destination address and has no frame check sequence; a `vsock` frame is a virtio-vsock packet, header first (Virtio 1.2 §5.10).
- Driver rules: one queue pair per CPU when offered, spread by the device's receive hash; one MSI-X vector per queue pair, bound to its CPU. After a receive interrupt the driver masks that vector and polls the queue until it is empty, then unmasks it; it uses notification suppression (virtio `EVENT_IDX`) when offered. Never suspended.

### Tpm

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `random` | `count: u16` (1 to 64) | `Untrusted[Bytes]` | `device.io`, `device.timeout` | `none` |
| `pcr_read` | `pcr: u8` | `Untrusted[Bytes]`, its 32-byte SHA-256 value | `device.out-of-range`, `device.io`, `device.timeout` | `none` |
| `nv_read` | `index: u32`, `offset: u16`, `count: u16` | `Untrusted[Bytes]` | `device.out-of-range`, `device.io`, `device.timeout` | `none` |
| `seal` | `secret: Bytes` (at most 128 bytes), `pcrs: u32` | `Bytes`: the sealed blob | `device.too-large`, `device.io`, `device.timeout` | `none` |
| `unseal` | `blob: Bytes` | `Untrusted[Bytes]`: the secret | `device.policy`, `device.io`, `device.timeout` | `none` |

- `seal` binds `secret` to the TPM's storage hierarchy and to the current SHA-256 values of the PCRs `pcrs` selects (none: to the TPM alone); `unseal` fails with `device.policy` once they differ.
- Driver rules: the CRB interface at locality 0, one command at a time, completion polled from 50 µs between polls doubling up to 10 ms; a command not done in 120 s fails with `device.timeout`; never suspended.

## DMA and IOMMU

On a machine with `iommu`, the core enables DMA and interrupt remapping before `devices` starts ([start sequence](04-kernel.md#start-sequence)), and these rules hold.

- Functions the IOMMU cannot tell apart, sharing a requester id or behind a PCIe-to-PCI bridge, form one **IOMMU group** with one IOMMU domain. A group's functions bind only to drivers of one principal and one [provenance class](09-authority.md#provenance); another binding gives `failed(device.group-conflict)`.
- A driver's DMA allocator draws buffers within the driver's memory limit and maps them into its group's domain; the bus address of an `hw::Dma` or `hw::DmaShared` buffer is its I/O virtual address (requires [N13](22-nan.md#n13)).
- Buffers of internal devices are mapped once, when allocated, with 2 MiB IOMMU pages where available. Buffers of external devices (below a port the firmware marks external-facing or hot-pluggable) are mapped at submission and unmapped at completion, whose IOTLB invalidation ends before the buffer returns.
- Interrupt remapping limits a device's MSI writes to its own vectors. PCIe address translation services are disabled on every function.
- A DMA access outside its domain is an IOMMU fault: the core emits `device.dma-fault` and abandons the driver's domain from outside (requires [N2](22-nan.md#n2)).

On a machine without `iommu`, bus addresses are physical, every driver's record shows `dma: unconfined`, and drivers bind only under the device rule of [placement](09-authority.md#placement).

## Interrupts and polling

- `devices` requests a device's vectors from the core ([vectors](04-kernel.md#vectors)): with MSI-X, one per queue, at most one per CPU plus one; else one MSI vector; else its INTx line as `_PRT` or the device tree routes it, level-triggered and possibly shared. A platform device gets its firmware-described lines. Each MSI-X vector targets one CPU (requires [N21](22-nan.md#n21)).
- A driver in the shared space installs an interrupt fn on each vector with `hw::IrqShared.install` ([NaN 24 §Locks](https://github.com/nan-language/nan/blob/main/docs/spec/24-kernel.md#locks)). The interrupt fn quiets the device and posts an `hw::Signal` ([NaN 24 §Signals](https://github.com/nan-language/nan/blob/main/docs/spec/24-kernel.md#signals)); a keyed task of the driver waits on the signal and sends the driver a message. A driver in an isolated space receives its interrupts as doorbells ([interrupts and vectors](04-kernel.md#vectors)), so it binds only to a device with MSI or MSI-X.
- A shared INTx line stays masked until the interrupt fn of every driver sharing it has run (requires [N21](22-nan.md#n21)).
- A vector raising more than 100,000 interrupts in 1 s is masked, `device.interrupt-storm` is emitted, and the driver is cut off and restarted.
- A device with no request pending raises no interrupt except to deliver input: `Console` bytes, `Nic` frames and link changes.

## Device power

A device is `on` (PCI D0), `suspended` (D3hot through its power management capability, or the low-power state of a platform device) or `off` (no driver, D3hot). Its binding's `PowerPolicy` drives these transitions; under `On` the device is never suspended:

| State | Event | Next | Effect |
| --- | --- | --- | --- |
| `off` | a driver binds to it under `Auto` or `On` ([driver instances](#driver-instances)) | `on` | — |
| `on` | under `Auto`, the protocol allows runtime suspend and the driver has had no request for the protocol's idle time | `suspended` | the runner calls the driver's `suspend()` ([system interfaces](06-execution.md#system-interfaces)), which quiets the device, then `devices` puts the device in its low-power state; `device.suspended` |
| `suspended` | a request to the driver, or the policy becomes `On` | `on` | `devices` powers the device on, then the runner calls `wake()` and delivers the waiting request, if any; `device.resumed` |
| `on`, `suspended` | the policy becomes `Off`, or the device is unbound | `off` | the driver stops and the device is cut off ([reset](#reset)) |

Power off and reboot use `\_S5`, which `devices.aml` evaluates, and the FADT reset register ([power off and reboot](08-scheduling.md#power-off-and-reboot)).

## Reset

**Cut-off** ends a device's access to memory and interrupts. When a driver's domain is abandoned (a trap, a CPU exception, the handler time limit, a DMA fault or an interrupt storm), these steps run before [settlement](https://github.com/nan-language/nan/blob/main/docs/spec/11-errors-domains.md#settlement) frees its memory (requires [N13](22-nan.md#n13)):

1. The core detaches the group's IOMMU domain on a machine with `iommu`, or clears the function's bus master bit; the device's DMA then faults or stops. `device.cut-off` is emitted with the reason.
2. It masks the device's vectors and uninstalls the driver's interrupt bindings.
3. It resets the function by function level reset when supported. A function unresponsive 1 s after its reset gets `failed(device.reset-failed)` and stays cut off.
4. It acknowledges every device operation the driver had in flight ([NaN 17 §Acknowledged cancellation](https://github.com/nan-language/nan/blob/main/docs/spec/17-suspension-io.md#acknowledged-cancellation)), and settlement frees the driver's DMA memory.
5. Every request pending at the driver fails with `device.reset`, not `port.server-stopped` ([trap handling](06-execution.md#trap-handling)).
6. The runner restarts the driver after the [restart](06-execution.md#restarts) backoff. Past the restart intensity the driver is `failed` and its failure does not escalate to `devices`; the device gets `failed(device.driver-failed)`, and `device.failed` is emitted.

Unbinding and the policy `Off` stop the driver at quiescence ([lifecycle](06-execution.md#lifecycle)), then run steps 1 to 3. A hot-plug removal runs steps 1, 2, 4 and 5, its pending requests failing with `device.unplugged`. The `reset` operation runs steps 1 to 6 without the backoff.

## Device classes

| Class | Protocol | Hardware | Implemented by |
| --- | --- | --- | --- |
| serial | `Console` | 16550, PL011 | entries `uart`, `pluart` |
| RTC | `Rtc` | CMOS RTC, PL031 | entries `cmos`, `plrtc` |
| entropy | `Entropy` | virtio-rng | entry `virtio-rng` |
| entropy | — | RDSEED | the core's random pool ([entropy](03-boot.md#entropy)) |
| entropy, TPM | `Tpm` | TPM 2.0, CRB interface | entry `tpm-crb` |
| block | `Block` | AHCI, NVMe, virtio-blk | entries `ahci`, `nvme`, `virtio-blk` |
| NIC | `Nic` | virtio-net, virtio-vsock | entries `virtio-net`, `virtio-vsock` |
| interrupts, IOMMUs, timers, watchdog | — | local APIC, x2APIC, IOAPIC, GICv3 and ITS, VT-d, AMD-Vi, SMMUv3, HPET, ACPI PM timer, Intel TCO, ACPI WDAT, SP805 | the core ([core](04-kernel.md#core), [watchdog](04-kernel.md#watchdog)) |

## The fat component

`fat` is the base instance that reads and writes the boot volume through the `boot-volume` role, for `boot` and `devices`. It reads FAT12, FAT16 and FAT32 with long names (Microsoft FAT32 File System Specification 1.03), writes long names with generated short names, and compares names without regard to case. Paths are boot volume paths such as `/nanos/boot.rec`.

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `list` | `path: text` | list of `FatEntry { name: Untrusted[text], dir: bool, size: u64 }` | `device.not-found`, `device.bad-volume`, `device.io` | `none` |
| `read` | `path: text`, `offset: u64`, `count: u32` | `Untrusted[Bytes]` | `device.not-found`, `device.out-of-range`, `device.bad-volume`, `device.io` | `none` |
| `write` | `path: text`, `data: Bytes` | — | `device.no-space`, `device.in-use`, `device.bad-volume`, `device.io` | `irreversible{erase}` |
| `overwrite` | `path: text`, `offset: u64`, `data: Bytes` | — | `device.not-found`, `device.out-of-range`, `device.in-use`, `device.bad-volume`, `device.io` | `irreversible{erase}` |
| `remove` | `path: text` | — | `device.not-found`, `device.in-use`, `device.bad-volume`, `device.io` | `irreversible{erase}` |
| `allocate` | `path: text`, `bytes: u64` | — | `device.exists`, `device.no-space`, `device.bad-volume`, `device.io` | `irreversible{actuate}` |
| `extent` | `path: text` | `first: u64`, `blocks: u64` (volume blocks) | `device.not-found`, `device.not-contiguous`, `device.bad-volume` | `none` |

- `write` creates or replaces a whole file, creating missing directories: it writes the data to free clusters, flushes, rewrites the file's directory entry in one block write, flushes, then frees the old clusters in both FATs. A crash leaves the old file or the new one, and clusters no entry reaches, which `fat` frees when it mounts the volume.
- `overwrite` replaces bytes inside the file's size in place, one aligned block being one device write; the [boot record](03-boot.md#boot-record) copies use it. `remove` deletes a file or an empty directory.
- `allocate` creates a file of `bytes` bytes on one run of free clusters, its contents unspecified; without such a run it fails with `device.no-space`. `extent` gives the block range of a file whose clusters are contiguous. While `devices.store-img` exists, `write`, `overwrite` and `remove` of `/nanos/store.img` fail with `device.in-use`.
- `fat` validates the boot sector and FAT geometry when it mounts the volume; after a failed validation every operation fails with `device.bad-volume`. It caches up to 256 KiB of FAT and directory blocks, dropped under memory [pressure](05-memory.md#pressure).

## Events

| Kind | Payload | Section |
| --- | --- | --- |
| `device.added`, `device.removed` | bus path, ids | [Discovery](#device-discovery) |
| `device.table-invalid` | table signature | [Discovery](#device-discovery) |
| `device.bound` | bus path, entry, driver moniker | [Drivers](#driver-instances) |
| `device.unbound` | bus path, driver moniker | [Manager](#the-device-manager) |
| `device.cut-off` | bus path, reason (`abandoned`, `stopped`, `removed`, `reset`) | [Reset](#reset) |
| `device.failed` | bus path, code | [Reset](#reset) |
| `device.dma-fault` | bus path, I/O virtual address, `read` or `write` | [DMA](#dma-and-iommu) |
| `device.interrupt-storm` | bus path, vector | [Interrupts](#interrupts-and-polling) |
| `device.suspended`, `device.resumed` | bus path | [Power](#device-power) |

## Codes

| Code | Meaning | Section |
| --- | --- | --- |
| `device.unknown` | no device has the bus path | [Manager](#the-device-manager) |
| `device.no-driver` | the device has no driver | [Manager](#the-device-manager) |
| `device.bar-unassigned` | the firmware left a memory BAR unassigned | [Discovery](#device-discovery) |
| `device.aml` | an AML evaluation the device needs failed | [Discovery](#device-discovery) |
| `device.driver-mismatch` | the named entry matches none of the device's identifiers | [Matching](#matching) |
| `device.config-protected` | a configuration write to a protected register | [Drivers](#driver-instances) |
| `device.no-vectors` | no vector is left for the device | [Drivers](#driver-instances) |
| `device.start-timeout` | the driver gave no `Health` reply within 5 s | [Drivers](#driver-instances) |
| `device.unhealthy` | the driver trapped or replied `Unhealthy` at start | [Drivers](#driver-instances) |
| `device.role-unresolved` | no device holds the role a wiring names | [Roles](#volumes-and-roles) |
| `device.not-contiguous` | the store file's clusters are not contiguous | [Roles](#volumes-and-roles) |
| `device.group-conflict` | a binding mixing principals or provenance classes in one IOMMU group | [DMA](#dma-and-iommu) |
| `device.reset` | the request was pending when its driver was cut off | [Reset](#reset) |
| `device.unplugged` | the request was pending when its device was removed | [Reset](#reset) |
| `device.reset-failed` | the function did not respond after its reset | [Reset](#reset) |
| `device.driver-failed` | the driver passed its restart limit | [Reset](#reset) |
| `device.io` | the device reported an error, carried in the failure record | [Protocols](#device-protocols) |
| `device.timeout` | the request outlived its protocol's time limit | [Protocols](#device-protocols) |
| `device.out-of-range` | an argument outside the unit, view, file or hardware range | [Protocols](#device-protocols) |
| `device.too-large` | data above the operation's limit | [Protocols](#device-protocols) |
| `device.unsupported` | the device lacks the operation | [Protocols](#device-protocols) |
| `device.unaligned` | partial blocks, or a zoned write off its write pointer | [Block](#block) |
| `device.read-only` | a write to a read-only unit | [Block](#block) |
| `device.link-down` | `send` while the link is down | [Nic](#nic) |
| `device.policy` | `unseal` while the PCR values differ from the sealed ones | [Tpm](#tpm) |
| `device.not-found` | no file or directory at the boot volume path | [fat](#the-fat-component) |
| `device.no-space` | the boot volume has no free cluster | [fat](#the-fat-component) |
| `device.in-use` | the file backs `devices.store-img` | [fat](#the-fat-component) |
| `device.exists` | a file already exists at the path | [fat](#the-fat-component) |
| `device.bad-volume` | the boot volume's FAT structures failed validation | [fat](#the-fat-component) |
