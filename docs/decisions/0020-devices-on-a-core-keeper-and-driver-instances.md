# 0020. Devices on a core keeper and driver instances

Status: accepted

## Context

Chapter 15 has `devices` hold every device of the machine and bind each to a driver instance of its own, which reaches only its device and whose trap cuts the device off before its memory is freed. NaN v0.6.0 gives a bare kernel's first task its PCI functions and ports through `boot::Kernel`. A function is linear and a resource of the domain run holding it: its windows, interrupts, power and reset derive from it without a trusted premise, and once a keeper is given to it, its release or its holder's settlement hands it back to that keeper cut off. Its INTx line is the only I/O APIC pin a kernel installs on, so an ISA device's line is out of reach. A component's Env may be linear, a handler may postpone a request and answer it from a later handler through its runner capability, and a server answers the calls of any protocol whose operations its own schema serves. NaN gives a register access the class `irreversible(emit, actuate, erase, spend)` and cannot yet seal a lower one (N15). The store, which holds `/sys/instances`, `/sys/grants`, `/sys/config` and `/live`, comes in later tasks.

## Decision

- **Packages.** `devices` holds the device protocols, bus paths, the `drivers` registry and its matching, partition tables, the AML evaluator, the volume view and the `devices` component; `drivers` holds one module per driver family; `fat` is the `fat` component. Each driver contributes its `drivers` entry.
- **The core keeper.** The kernel's first task gives every PCI function a keeper and holds it with its keeper, every port, and copies of the ACPI tables `devices` reads. The device root is the kind `devices::Root`, whose orders the keeper serves: the devices and their identifiers, a table, the platform devices `devices` declares from the AML namespace with their port windows, the cut-off, reset and power of a device, the topology's changes, and the device events, which it writes to the core rings. Only the core's root may claim a device.
- **Driver instances.** `devices` creates each driver and volume view through its `Control`: the runner adds the instance with its grants, a `Device` grant minting an `os::Device` claim on the bus path. The kernel serves a driver by claiming its device through the root and building the driver's linear Env from the function or platform windows; the Env's abandonment releases them, and a trapped driver's settlement hands its function back, cut off, for the next claim to reclaim.
- **Requests.** Each driver serves a protocol of its own that adds private operations, such as completions, to its device protocol's, and answers calls made under the device protocol. A request its device completes later is postponed and answered from the completion's handler. Each operation that reaches a register declares the class NaN gives a register access.
- **Interrupts and DMA.** PCI drivers take MSI-X, else MSI, interrupts on CPU 0 through their function, an interrupt fn posting a signal that a keyed task turns into a message. Platform UARTs and the RTC are polled. Functions are attached to physical memory and their buffers drawn in the physical space: DMA is unconfined until isolation comes.
- **Before the store.** Driver and view specifications and their grants live in the runner, roles in its directory, which resolves a wire naming a role; `/live/devices` records live in the state of `devices`, and configuration is the defaults. `supervisor` starts `fat` once `devices` has settled the role holders the start order needs.
- **The log UART.** Until `journal` runs, the core writes its event log to the 16550 at port `0x3F8`, so `devices` binds no driver to it and the `console` role resolves among the other UARTs.

Alternatives rejected:

- **`devices` holding the functions:** a driver's function would have to leave the domain of `devices` for the driver's, and nothing would hand it back when the driver traps.
- **Drivers serving the device protocols themselves:** a component's messages are its protocol's operations, so a completion would have to be a public operation every client could call.
- **Mapping the platform UARTs' and RTC's lines through the I/O APIC directly:** NaN's runtime owns its pins.

## Consequences

- The device protocols' declared classes are higher than chapter 15's until NaN seals lower ones.
- Platform devices are polled and take no interrupt until NaN installs ISA lines.
- Device grants are trust escapes and DMA reaches all memory until isolation gives each group an I/O domain.
- Bindings, specifications and `/live/devices` records do not outlive a boot until the store holds them.
