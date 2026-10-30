# The kernel

This chapter defines the kernel image, the core and its modules, the start sequence, CPUs, the anchor, vectors, address spaces, placement mechanics, user entry, kernel policies, faults, kernel halts, the watchdog, and the `fault` family.

## Kernel image

The **kernel image** is `kernel.elf`: one NaN final build of the program `kernel` for `x86_64-bare`, or `aarch64-bare` on AArch64. Its closure holds the core and the component packages of every [base instance](20-base.md#base-instances). [NaN 27 §Images](https://github.com/nan-language/nan/blob/main/docs/spec/27-program-model.md#images) fixes its format and the physical addresses of its segments. The [loader](03-boot.md#loader) places it and enters it with the [handoff record](03-boot.md#handoff-record) and the genesis record.

- Domains and components run in the kernel image (requires [N2](22-nan.md#n2), [N3](22-nan.md#n3)).
- The components of the base instances form one [bundle](10-code.md#artefacts-and-bundles), so values move between base instances without encoding.
- The image's registry tables are the initial entries of the load-time [registries](10-code.md#registries), among them [`drivers`](15-devices.md#drivers) and the [kernel policies](#kernel-policies) (requires [N6](22-nan.md#n6)).
- [21-performance](21-performance.md) bounds its size.

## Core

The **core** is the kernel code that is not a component: the `kernel` package. Its tasks, interrupt fns and landing fns run in NaN's kernel domain, so a trap anywhere in the core is a [kernel halt](#kernel-halts).

| Module | Holds | Rules |
| --- | --- | --- |
| `cpu` | per-CPU records, CPU features, topology, idle entry | [CPUs](#cpus), [idle](08-scheduling.md#idle) |
| `frames` | the frame allocator, the core reserve, the pools | [frames](05-memory.md#frames), [pools](05-memory.md#pools) |
| `spaces` | page tables, PCIDs and ASIDs, code areas, TLB shootdowns, split tables | [Address spaces](#address-spaces) |
| `irq` | device vectors, I/O APIC entries, MSI and MSI-X messages, GIC routes and ITS mappings | [Vectors](#vectors) |
| `iommu` | IOMMU units, device domains, interrupt remapping | [DMA and IOMMU](15-devices.md#dma-and-iommu) |
| `time` | clock sources, timer heaps, the watchdog | [clocks](08-scheduling.md#clocks), [timers](08-scheduling.md#timers), [Watchdog](#watchdog) |
| `sched` | run queues, classes, accounts, policy calls | [08-scheduling](08-scheduling.md) |
| `domains` | domain runs, settlement, abandonment from outside | [Faults](#faults) |
| `caps` | capability tables, the OS capability kinds, revocation | [capability kinds](09-authority.md#capability-kinds) |
| `runner` | intakes, handler calls, quiescence, checkpoints, hibernation, forks | [runner](06-execution.md#runner) |
| `load` | loading and unloading bundles, registry entries | [loading](10-code.md#loading) |
| `journal` | per-CPU event rings | [events](17-observability.md#events) |
| `entry` | the system call and user-exception landing fns | [User entry](#user-entry) |

- The core reads the handoff record, the genesis record, firmware tables and every value an isolated space gives it only through validators ([NaN 15 §Validators](https://github.com/nan-language/nan/blob/main/docs/spec/15-untrusted.md#validators)).
- The core holds no policy table. Choices beyond the rules of this specification are [kernel policies](#kernel-policies).
- Every trust escape of the core is listed in its ledger and carries a signed seal by a `vendor.*` or `human.*` principal other than its author (requires [N15](22-nan.md#n15)).

## Start sequence

The kernel starts in these steps. A step that fails halts the kernel with its code ([Kernel halts](#kernel-halts)).

1. **Handoff.** `main(take k: boot::Kernel)` starts on the boot CPU ([NaN 27 §Machine runs](https://github.com/nan-language/nan/blob/main/docs/spec/27-program-model.md#machine-runs)), reads the handoff record and the genesis record where the start information or the device tree places them (requires [N14](22-nan.md#n14)), or the genesis record the image holds when none is passed, and decodes both through their validators. A passed genesis record's hash equals the one the handoff record carries. The handoff record's entropy goes to the core's random source ([entropy](03-boot.md#entropy)). Failure: `fault.handoff-invalid`.
2. **CPU features.** The core reads the boot CPU's identification (requires [N11](22-nan.md#n11)) and determines the [machine conditions](01-machines.md#machine-conditions) `pages-1g`, `pcid`, `invariant-tsc`, `tsc-deadline`, `mwait` and `meltdown-immune`. Failure: none; an absent condition selects its fallback.
3. **Frames.** The core builds the [frame allocator](05-memory.md#frames) from `k.regions()`, sets aside the [core reserve](05-memory.md#core-reserve) and makes the node [pools](05-memory.md#pools). Failure: `fault.memory-insufficient` when usable memory is below the [minimum machine](01-machines.md#minimum-machine)'s or cannot hold the core reserve.
4. **Shared space.** The core builds the tables of the [shared space](#shared-space) and installs them on the boot CPU with `write_cr3`, `write_ttbr0` on AArch64. Failure: `fault.memory-insufficient`.
5. **Firmware tables.** The core decodes the ACPI tables (MADT, FADT, MCFG, SRAT, SLIT, HPET, DMAR or IVRS, WDAT) or the device tree: CPUs, interrupt controllers, NUMA nodes and their distances, timers, power and reset registers, IOMMUs, the watchdog, and the PCIe configuration windows, which go to [`devices`](15-devices.md#device-discovery). Failure: `fault.firmware-tables-invalid` when the MADT or the FADT on x86_64, or the device tree's `memory`, `cpus` or interrupt controller nodes on AArch64, are missing or fail their validators. Any other table that fails its validator is ignored, and the conditions only it reports count as absent.
6. **Clock.** The core calibrates the [monotonic clock](08-scheduling.md#monotonic-clock). Failure: none.
7. **Watchdog.** On a machine with `watchdog`, the core arms the [watchdog](#watchdog). Failure: none; a watchdog that does not respond counts as absent.
8. **IOMMU.** On a machine with `iommu`, the core enables every IOMMU unit with all DMA blocked and interrupt remapping on; `devices` gives each device group a domain of its own when it binds a driver ([DMA and IOMMU](15-devices.md#dma-and-iommu)). Failure: the boot continues as on a machine without `iommu`, `/live/security` reports `dma: unconfined`, and the core emits `fault.iommu-failed`.
9. **Anchor.** The core calls `take k.anchor(take state, run: first)` with the [anchor](#anchor) ([NaN 24 §Anchor](https://github.com/nan-language/nan/blob/main/docs/spec/24-kernel.md#anchor)). NaN starts every CPU, and each CPU enables its features ([CPUs](#cpus)) and loads the shared space's tables. Failure: NaN's halts.
10. **First task.** The first task starts `supervisor`, which starts the base instances in the [start order](20-base.md#start-order), in parallel where independent and lazily where marked, and then reconciles `/sys/instances` ([lifecycle](06-execution.md#lifecycle)). The first task never returns; its end halts the kernel with NaN's cause `first-task-ended`.

## CPUs

- The machine's CPUs are those the MADT or the device tree reports usable. NaN numbers them from 0, the boot CPU being 0. The set is fixed for the boot.
- As the anchor starts it, each CPU enables no-execute pages, write protection in kernel mode, SMEP, SMAP and UMIP when it has them, PCIDs on a machine with `pcid`, and x2APIC mode when it has it (requires [N11](22-nan.md#n11), [N21](22-nan.md#n21)); on AArch64, privileged access never and privileged execute never for user pages.
- The core records each CPU's package, core, sibling hardware threads, last-level cache and NUMA node, from CPUID leaves 4, 0xB and 0x1F and the SRAT, or from the device tree's `cpu-map`, cache nodes and `numa-node-id` properties. [Wake placement](08-scheduling.md#wake-placement) uses them, and `/live/cpus` publishes them ([`/live`](13-tree.md#live)).
- Wake IPIs and TLB-shootdown IPIs are services of NaN's runtime (requires [N8](22-nan.md#n8), [N11](22-nan.md#n11)); the core sends no other IPI.

## Anchor

The anchor holds exactly what entry contexts (interrupt fns and landing fns) need. The core's tasks reach the rest of its state through what the first task captured.

| Part | Holds | Used by |
| --- | --- | --- |
| `cpus` | per CPU, in an `hw::PerCpu`: its number, node, cache and sibling ids; the running domain, account and space; the [accounting](08-scheduling.md#accounting) accumulator; the [timer](08-scheduling.md#timers) heap; the journal ring; the idle state | every entry context |
| `spaces` | per isolated space: its roots, its tag on each CPU, the mappings of its user window, its domains, handles and doorbells | `entry` landing fns, shootdowns |
| `routes` | per device vector of a driver in an isolated space, the driver's doorbell | the core's device interrupt fn |
| `policies` | the active entry of each kernel policy and its overrun count | scheduling and idle paths |
| `watchdog` | the watchdog's registers and state | timer and idle paths |

Each mutable part sits behind an `hw::SpinLock`, an `hw::PerCpu` or atomics.

## Vectors

NaN's runtime owns each CPU's interface to its interrupt controller: the local APIC or x2APIC and its end of interrupt, the GIC CPU interface, the CPU's timer, the IPIs and the spurious vector ([NaN 24 §Interrupt functions](https://github.com/nan-language/nan/blob/main/docs/spec/24-kernel.md#interrupt-functions)). The core binds the exception vectors and routes device interrupts.

| Vectors | x86_64 | AArch64 | Owner |
| --- | --- | --- | --- |
| exceptions | 0 to 31 | 0 to 15 | the core ([Faults](#faults)) |
| timer, IPIs, spurious | the runtime's | the virtual timer, the SGIs | NaN's runtime |
| devices | every other vector from 32 | every other interrupt from 32, and LPIs | the core's `irq` |

- `irq` allocates each device vector that `devices` requests ([interrupts and polling](15-devices.md#interrupts-and-polling)) on one CPU (requires [N21](22-nan.md#n21)): the CPU named for a per-CPU queue, or else the CPU of the device's NUMA node holding the fewest device vectors.
- It programs the source: an I/O APIC redirection entry, an MSI or MSI-X message, a GICv3 route or an ITS mapping.
- A driver in an isolated space receives only message-signalled interrupts: the core's interrupt fn rings the driver's doorbell from `routes` and completes the interrupt (requires [N23](22-nan.md#n23)).

An interrupt costs one seam entry; an idle device costs nothing.

## Address spaces

A **space** is an address space: the **shared space**, whose code runs in ring 0 (EL1 on AArch64), or an **isolated space**, whose code runs in ring 3 (EL0).

### Shared space

| Range | Mapped at | Attributes |
| --- | --- | --- |
| usable memory | its physical address | read-write, non-executable, supervisor-only, global; 1 GiB pages on a machine with `pages-1g`, else 2 MiB pages, and 4 KiB pages at range edges |
| the kernel image | its physical address | code read-execute, read-only data read-only, other data read-write and non-executable |
| code areas | the physical addresses of their frames, replacing the usable-memory mapping | code read-execute, read-only data read-only |
| MMIO windows of the core and of shared drivers | their physical addresses | uncached, non-executable, supervisor-only |
| holes between ranges | nothing | — |

- The shared space maps every frame and region at its physical address ([NaN 24 §Physical memory](https://github.com/nan-language/nan/blob/main/docs/spec/24-kernel.md#physical-memory)), whatever the amount of memory (requires [N12](22-nan.md#n12)).
- A **code area** holds the code and read-only data of a loaded bundle ([loading](10-code.md#loading)); every space that loads the bundle maps the same frames. Loading writes them through the usable-memory mapping, shoots down every CPU's translations of them, then maps them as the table gives, so no writable mapping of them remains. Unloading maps them read-write and non-executable again after a shootdown.

### Isolated spaces

- An isolated space has a root of its own. Below its user window it shares the shared space's tables, so the core's mappings are present and supervisor-only (requires [N23](22-nan.md#n23)).
- The **user window** starts at B, the first 512 GiB boundary above the highest physical address the shared space maps, and ends at 2^47 on x86_64 and at 2^48 on AArch64. A machine whose B lies above 2^46 lacks `user-mode`.
- The window holds the code areas of the space's bundles (user, read-execute or read-only), its domains' chunks ([mappings](05-memory.md#mappings)), mapped blobs (read-only), its doorbell frame, the frames of its [channels](07-communication.md#isolated-transport) and the MMIO windows of its drivers (uncached). Nothing in it is both writable and executable.
- The shared space uses PCID 0, ASID 0 on AArch64. Each CPU gives tags 1 to 63 to the isolated spaces it ran most recently, and reassigns the least recently used tag after invalidating its translations. On a machine without `pcid`, every switch between spaces flushes the CPU's non-global translations.

### Split tables

On a machine without `meltdown-immune`, each isolated space has a second root, its user root, mapping only the user window and the entry trampoline (requires [N23](22-nan.md#n23)). A CPU runs user code on the user root, switches to the kernel root at every entry to the core, by system call, exception or interrupt, and back at every return. On a machine with `pcid`, the user root's PCID is the kernel root's plus 2,048, so the switch flushes nothing. [Costs of isolation](09-authority.md#costs-of-isolation) gives the price of each crossing.

### Switches and shootdowns

- Switching a CPU to a space loads its root with the CPU's tag for it. Before running an isolated space of a principal other than the last principal whose isolated code ran on that CPU, the core issues an indirect branch prediction barrier; on CPUs affected by data-sampling flaws, it clears the CPU buffers before every return to user mode (requires [N11](22-nan.md#n11)).
- Removing a mapping completes only when no CPU can use it: CPUs running the space take a shootdown IPI and invalidate the range, and CPUs holding a tag for the space without running it invalidate that tag before they run the space again (requires [N11](22-nan.md#n11)). A frame whose mapping is removed returns to the frame allocator only after the removal completes. A change to the shared space shoots down every CPU.
- A switch costs one root load; a shootdown costs one IPI round to the CPUs running the space; an idle space costs nothing.

## Placement mechanics

[Placement](09-authority.md#placement) decides, when an instance starts, whether it runs in the shared space or in an isolated space; the core carries the decision out.

- An instance placed shared runs its tasks in ring 0.
- An instance placed isolated runs in the isolated space that placement names. The core creates an isolated space when an instance first starts in it, and destroys it, freeing its tables, when its last instance has stopped.
- The runner loads a bundle into each space where an instance of it starts ([loading](10-code.md#loading)).
- An instance moves to another space only by a restart there ([provenance](09-authority.md#provenance)).
- On a machine without `user-mode`, the core creates no isolated space ([machine conditions](09-authority.md#machine-conditions)).

## User entry

Code of an isolated space runs on **space threads**: kernel threads that `sched` runs as it runs tasks ([run queues](08-scheduling.md#run-queues)) and that enter user mode on the space's root with a saved register state. A space has at most one space thread per CPU, on which NaN's user-mode runtime runs the space's tasks and handlers (requires [N23](22-nan.md#n23)).

- A space thread leaves user mode by the system call seam, an exception or an interrupt, whose landing fns are those of `entry` ([NaN 24 §User exceptions and syscalls](https://github.com/nan-language/nan/blob/main/docs/spec/24-kernel.md#user-exceptions-and-syscalls)). A timer interrupt preempts a space thread as it preempts a task.
- While a space thread runs user code, the core denies user mode the time-stamp counter (CR4.TSD) or the generic counter (CNTKCTL_EL1) (requires [N11](22-nan.md#n11)), so user code reads time only through `now`, at its clock's [resolution](08-scheduling.md#resolution).
- Each space has a **doorbell frame**, shared with the core, holding one bit per doorbell of the space, at most 32,768: the core sets a bit when its doorbell rings, and the runtime clears it. The core gives a doorbell to each side of a channel the space holds and to each vector of the space's drivers.
- A **handle** is a capability of an instance of the space, named by its slot in the space's handle table, which the core keeps (requires [N27](22-nan.md#n27)).
- Every exception raised in user mode is an [integrity fault](#integrity-faults).

### System calls

A system call passes its number and up to six arguments in the registers NaN 24 names, each read as an `Untrusted[u64]` through a validator. A result from 2^64 − 4,096 up is a failure: 2^64 − 1 is `invalid`, 2^64 − 2 `stale` and 2^64 − 3 `exhausted`. Every other result is a success. An unknown number fails with `invalid`.

| Number | Call | Arguments | Result | Failures |
| --- | --- | --- | --- | --- |
| 0 | `run` | domain slot | 0 | `stale`: no domain of the space has the slot |
| 1 | `wait` | deadline in monotonic ns, or 0 for none | 1 once a bit of the doorbell frame is set, 0 at the deadline | — |
| 2 | `ring` | channel | 0 | `invalid`: the space holds no side of the channel; `stale`: the channel is closed |
| 3 | `draw` | domain slot, chunk count | the window address of the chunks | `stale`, `exhausted` |
| 4 | `release` | window address, chunk count | 0 | `invalid`: chunks the domain does not hold |
| 5 | `now` | — | monotonic ns at the resolution of the running domain's clock | — |
| 6 | `invoke` | handle, operation, window address and length of the arguments, window address and capacity of the result | the result's length | `stale`, `invalid`, `exhausted` |
| 7 | `transfer` | channel, window address, length | a transfer number | `invalid`, `exhausted` |
| 8 | `accept` | channel, transfer number, domain slot | the window address of the payload | `invalid`, `exhausted` |
| 9 | `pass` | channel, handle | a capability id pending for the channel | `stale`, `invalid` |
| 10 | `claim` | channel, pending capability id, domain slot | a handle | `invalid` |

- `run` names the domain whose code the thread runs next: its account is charged for the thread ([accounting](08-scheduling.md#accounting)), and its domain answers for an exception. A space thread runs the domain of the instance the core started it for as long as it has not called `run`.
- `wait` blocks the thread while no bit of the doorbell frame is set, at most to the deadline; a waiting space thread costs nothing.
- `draw` draws chunks through the domain's cell and maps them; `release` unmaps and returns them ([mappings](05-memory.md#mappings)).
- `invoke` performs an operation that never waits of a kernel-implemented [capability kind](09-authority.md#capability-kinds) (requires [N27](22-nan.md#n27)); its arguments and result are `mem::wire` encodings, read and written by bounded copies (requires [N23](22-nan.md#n23)). An operation that waits reaches the instance serving it through a channel.
- `transfer`, `accept`, `pass` and `claim` are the core operations of the [isolated transport](07-communication.md#isolated-transport): `transfer` hands the core a payload for a channel; `accept` gives it to the receiving domain, copied into chunks drawn through the domain's cell or with its frames moved into the receiver's window; `pass` and `claim` move a capability.

## Kernel policies

A **kernel policy** is a choice the core makes by calling the function field of the active entry of a load-time registry ([NaN 25 §Function fields](https://github.com/nan-language/nan/blob/main/docs/spec/25-registries.md#function-fields)).

| Policy | Called | Inputs | Result | Default |
| --- | --- | --- | --- | --- |
| `sched.pick` | at each scheduling point of a CPU | the CPU's eligible entities, with class, weight and deadline | the entity to run | [CPU classes](08-scheduling.md#cpu-classes) |
| `sched.place` | at each wake | the previous CPU, the waker's CPU, the idle CPUs by cache and node | a CPU | [wake placement](08-scheduling.md#wake-placement) |
| `mem.reclaim` | at pressure level 3 | the instances that hibernation for pressure may take, with the time of their last message and their encoded size | the order of hibernation | [pressure](05-memory.md#pressure) |
| `power.idle` | when a CPU goes idle | the predicted idle time, the idle states with their exit latencies, the latency tolerance | an idle state | [idle states](08-scheduling.md#idle-states) |
| `power.perf` | when the classes a CPU runs change | those classes and the CPU's utilisation | an energy-performance preference | [frequency](08-scheduling.md#frequency) |

- The function field of each entry type demands `limits notrap, terminates, noalloc, stack N`. The kernel image contributes the default entry of each registry.
- The core calls a policy with interrupts disabled and passes it at most 256 candidates.
- Only the bundles that [registries](10-code.md#registries) names contribute a policy entry. The runner loads such a bundle into the shared space when the plan installing it applies; its entry replaces the active one as registries state, and the core calls it by N6's convention (requires [N6](22-nan.md#n6)).
- The core reads the clock before and after every call. A call longer than 5 µs, or one whose result names no candidate, is an overrun.

| State | Event | Next | Effect |
| --- | --- | --- | --- |
| default | a bundle's entry replaces the active one | contributed | that entry is active |
| contributed | the active entry's fourth overrun within 1 s | fallen back | the kernel image's entry is active; `fault.policy-overrun` |
| contributed | the contributing bundle unloads | default or contributed | the entry it replaced is active again |
| fallen back | a plan installs a bundle contributing to the policy | contributed | the new entry is active |
| fallen back | every bundle contributing to the policy unloads | default | — |

The registry of `sched.pick`, in the core's module `sched`, with its entry type; a candidate's key is its virtual deadline in `normal` and `background` and its deadline in `latency`:

```nan
pub closed struct Candidate derives Copy, Eq {
    pub entity: u32
    pub class: u8
    pub weight: u16
    pub key_ns: u64
} Candidate

pub closed struct PickPolicy {
    pub slot: Unit
    pub pick: fn(ready: Slice[Candidate], now_ns: u64) -> usize limits notrap, terminates, noalloc, stack 1_024
} PickPolicy

pub registry sched_pick: PickPolicy keyed by slot
```

## Faults

| Fault | Raised in | Outcome | Event |
| --- | --- | --- | --- |
| NaN trap | a task of an instance's domain, in any space | the domain is abandoned ([trap handling](06-execution.md#trap-handling)) | [06](06-execution.md#events) |
| CPU exception | a task of an instance's domain in the shared space | an [integrity fault](#integrity-faults) of that domain | `fault.integrity` |
| CPU exception | user mode of an isolated space | an integrity fault of the domain the space thread's last `run` named | `fault.integrity` |
| breach of the isolated transport | an isolated side of a channel | an integrity fault of the instance whose side wrote it | `fault.integrity` |
| trap or CPU exception | the boot context, a core task, an interrupt fn, a landing fn or a kernel policy | a [kernel halt](#kernel-halts) | — |
| machine check, double fault, NMI outside a halt | any context | a kernel halt | — |
| IOMMU fault | a device | the driver's domain is abandoned ([DMA and IOMMU](15-devices.md#dma-and-iommu)) | [15](15-devices.md#events) |
| handler past its time limit | a handler | the domain is abandoned from outside ([handler limits](06-execution.md#handler-limits)) | [06](06-execution.md#events) |

### Integrity faults

An **integrity fault** is a CPU exception raised by an instance's code, or a breach of the [isolated transport](07-communication.md#isolated-transport) by the instance's side of a channel. The core handles one in these steps:

1. It records the vector, the error code, the faulting address and the instruction address, or the channel and the field of a breach, and abandons the instance's domain from outside, with settlement ([NaN 11 §Abandoning a domain](https://github.com/nan-language/nan/blob/main/docs/spec/11-errors-domains.md#abandoning-a-domain)) (requires [N2](22-nan.md#n2)).
2. It writes the fault report as the trap report of the abandonment, which the instance's [dead letter](06-execution.md#dead-letters) keeps.
3. It emits `fault.integrity` with severity `critical`.
4. The runner applies the instance's [restart policy](06-execution.md#restarts). The bundle stays admitted: no fault bans it.

A CPU exception during these steps halts the kernel.

The fault report has the form of a [NaN trap report](https://github.com/nan-language/nan/blob/main/docs/spec/11-errors-domains.md#trap-reports), with `fault.integrity` in place of the trap kind and the operand line `vector = v, code = c, address = a`, located through the bundle's line tables. An instruction outside every line table makes the first line `fault.integrity at 0x<address> in <bundle short hash>`; a breach makes it `fault.integrity at channel <n> in <moniker>`, with the operand line `field = f, value = v`.

```text
fault.integrity at ring/ring.nan:57 in ring::Ring.push
    self.slots[index] = entry
    vector = 14, code = 2, address = 0x7f3a_0000_1000
    called from ring/ring.nan:21 in ring::Ring.on
```

## Kernel halts

- The kernel halts on a trap or a CPU exception in the core and on NaN's other halt causes ([NaN 24 §Kernel halts](https://github.com/nan-language/nan/blob/main/docs/spec/24-kernel.md#kernel-halts)). The report goes to the serial console only; the link never prints a line starting with `trap.` or `halt ` ([framing](18-link.md#framing)).
- The core also halts the kernel at a failed [start](#start-sequence) step, at a failed store mount ([mounting and recovery](12-store-format.md#mounting-and-recovery)), at a failed commit of the base instances at boot ([genesis](13-tree.md#genesis)), when a base instance that the [start order](20-base.md#start-order) requires is not running in time, and when the reboot at the [health gate](03-boot.md#health-gate)'s deadline fails. Before NaN's report it prints one line `kernel: <code> <subject>` on the serial console: the step's code with the record or table at fault, as in `kernel: fault.firmware-tables-invalid MADT`; the mount's code; the commit's code with `generation <n>`; `fault.base-not-running` with the moniker; or `fault.power-failed` with the generation.
- What follows a halt is [reboot after a halt](03-boot.md#reboot-after-a-halt).
- At the next boot, `boot` compares the [boot record](03-boot.md#boot-record) with `/sys/boot`, its mirror of the last state it wrote. A `bad` slot, or a trial try consumed by an earlier boot, that the mirror lacks marks a boot that ended before its health gate, and `boot` emits `fault.previous-boot-unhealthy`.

## Watchdog

On a machine with `watchdog`, the core drives the hardware watchdog that the firmware tables describe: an ACPI WDAT table, an Intel TCO timer or an SP805. Its timeout is 30 s by default (`BootConfig.watchdog_s`, [health gate](03-boot.md#health-gate)).

| State | Event | Next | Effect |
| --- | --- | --- | --- |
| off | start step 7 | armed | counting from the timeout |
| armed | a third of the timeout since the last refresh | armed | refreshed |
| armed | the machine becomes [idle](08-scheduling.md#idle-machine) | stopped | counting stops |
| stopped | a CPU leaves idle | armed | refreshed and counting, before any task runs |
| armed | a kernel halt | expiring | no refresh: the machine resets within the timeout |
| stopped | a kernel halt, which only an interrupt with no binding causes on an idle machine | stopped | the machine stays stopped, as without `watchdog` |
| armed or stopped | [power off or reboot](08-scheduling.md#power-off-and-reboot) | off | stopped |
| off | a power off or reboot fails ([power off and reboot](08-scheduling.md#power-off-and-reboot) step 4) | armed | refreshed and counting |

The refresh is a `background` [timer](08-scheduling.md#timers) of the core. An idle machine has none, so the watchdog adds no wake-up.

## Events

| Kind | Payload | Section |
| --- | --- | --- |
| `fault.integrity` | moniker, bundle hash, vector, error code, faulting address or channel, fault report | [Integrity faults](#integrity-faults) |
| `fault.policy-overrun` | policy, bundle hash, overruns in the last second, longest call in ns | [Kernel policies](#kernel-policies) |
| `fault.iommu-failed` | IOMMU unit, reason | [Start sequence](#start-sequence) |
| `fault.previous-boot-unhealthy` | generation, slot, tries left, slot state | [Kernel halts](#kernel-halts) |

## Codes

| Code | Meaning | Section |
| --- | --- | --- |
| `fault.handoff-invalid` | the handoff record or the genesis record fails its validator, or the genesis record's hash differs from the handoff record's | [Start sequence](#start-sequence) |
| `fault.memory-insufficient` | usable memory is below the minimum, or cannot hold the core reserve and the shared space's tables | [Start sequence](#start-sequence) |
| `fault.firmware-tables-invalid` | a firmware table the start needs is missing or fails its validator | [Start sequence](#start-sequence) |
| `fault.integrity` | the instance's code raised a CPU exception, or its side of a channel breached the isolated transport | [Integrity faults](#integrity-faults) |
| `fault.base-not-running` | a base instance that the start order requires is not running in time | [Kernel halts](#kernel-halts) |
| `fault.power-failed` | the machine still runs 5 s after a power off or a reboot | [power off and reboot](08-scheduling.md#power-off-and-reboot) |
