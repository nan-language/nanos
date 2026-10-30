# Requirements on NaN

This chapter defines the features NaN OS requires of NaN beyond language 1 as the pinned toolchain v0.1.0 implements it. Each requirement states what language 1 provides, what NaN OS requires, the chapters whose rules depend on it, and the issue of the NaN repository that requests it. A rule elsewhere followed by `(requires Nn)`, linked to its entry here, depends on Nn. The identifiers are stable; N18 is unassigned.

## N1

**Targets for NaN OS.**

- **Language 1:** the targets `x86_64-nanos` and `aarch64-nanos` are reserved (`manifest.reserved-target`, NaN 37 §Reservations). Host entries return `Unit` or `u8` (NaN 27 §Entry). Machine targets reject floats (NaN 24 §Floating point).
- **NaN OS requires:**
  - **Descriptors and platform.** The two descriptors and a `nanos_platform` package. Images are lowered against the support routines the running kernel exports.
  - **Entries.** A component as entry, and the job entry `fn main(take world: nanos::World, take input: I) -> O fails E uses wait`, `I` and `O` implementing `Encode` and `E` implementing `Render`.
  - **Retargeting.** Host programs and libraries build for the nanos targets.
  - **Code.** Floats and `core::simd` are allowed.
  - **Memory.** Stacks the runtime grows for code without stack bounds, and nanos images whose boot heap is smaller than the 16 MiB of the machine targets.
  - **Interfaces.** The std interfaces `Resume`, `Pressure`, `Health` and `Suspend`.
- **Chapters:** [00](00-charter.md), [01](01-machines.md), [05](05-memory.md), [06](06-execution.md), [09](09-authority.md), [10](10-code.md), [11](11-store.md).
- **Issue:** [nan-language/nan#4](https://github.com/nan-language/nan/issues/4).

## N2

**Fault-contained domains in kernels.**

- **Language 1:** `sys::Domain.run` is `domain.unavailable` on machine targets (NaN 11 §Domain run), and any trap halts every CPU (NaN 24 §Kernel halts). Trap kinds are closed, and no operation abandons a domain from outside.
- **NaN OS requires:**
  - domain runs with settlement in kernels;
  - a CPU exception in a domain's code abandons that domain, under a trap kind of its own;
  - abandoning a domain from outside.
- **Chapters:** [04](04-kernel.md), [06](06-execution.md), [15](15-devices.md).
- **Issue:** [nan-language/nan#5](https://github.com/nan-language/nan/issues/5).

## N3

**A runner seam.**

- **Language 1:** components run only under `host::Runner`, which is `component.unavailable` on machine targets (NaN 26 §Runner). Only std's runner creates `sys::Cx`. Its only operations are `send`, `after`, `spawn` and `cancel` (NaN 26 §Component context, NaN 26 §Keyed tasks).
- **NaN OS requires:** a runner seam on machine and nanos targets. NaN OS's runner holds the state and the Env, mints a `sys::Cx` per handler call, decides when keyed tasks start, and encodes and decodes the state at quiescence. The `sys::Cx` it mints also has the operations `send_to`, `reply`, `publish`, `sync`, `capability`, `drop_capability`, `resource`, `take_resource` and `put_resource`, which NaN OS's runner implements.
- **Chapters:** [04](04-kernel.md), [06](06-execution.md), [07](07-communication.md), [14](14-changes.md).
- **Issue:** [nan-language/nan#6](https://github.com/nan-language/nan/issues/6).

## N4

**Protocols.**

- **Language 1:** the item words `protocol` and `ipc` are reserved (NaN 02 §Reserved forms). A component has one handler for all its messages, so all operations share one digest.
- **NaN OS requires:** `protocol` items with:
  - named operations, each with its head, contracts, digest and declared reversibility bound;
  - a protocol hash over the heads and the schema hashes of their types;
  - derived codecs between separately built closures;
  - an optional summary type that its servers publish;
  - capability fields transferred beside the encoding.
- **Chapters:** [06](06-execution.md), [07](07-communication.md), [11](11-store.md), [14](14-changes.md), [15](15-devices.md).
- **Issue:** [nan-language/nan#7](https://github.com/nan-language/nan/issues/7).

## N5

**Schema evolution.**

- **Language 1:** decoding an open type skips unknown fields (NaN 13 §Decoding rules), so re-encoding drops them. The item words `flexible` and `migrate` are reserved.
- **NaN OS requires:**
  - **`flexible` types:** decoders keep unknown fields, and encoders re-emit them.
  - **`migrate` functions:** a total conversion from an older shape, matched by its schema hash, with an optional inverse.
- **Chapters:** [10](10-code.md), [14](14-changes.md).
- **Issue:** [nan-language/nan#8](https://github.com/nan-language/nan/issues/8).

## N6

**Loading.**

- **Language 1:** `sys::load` is reserved (NaN 28 §Reserved std names). The machine code of one final closure belongs to no other image (NaN 30 §Lowering), and registry tables are fixed at the final build (NaN 25 §Tables).
- **NaN OS requires:**
  - lowering a closure against a running kernel's exported support routines;
  - loading it write-xor-execute, and unloading it;
  - calling function entries across closures through a defined convention;
  - registries whose entries loading adds, replaces and removes.
- **Chapters:** [04](04-kernel.md), [10](10-code.md), [15](15-devices.md).
- **Issue:** [nan-language/nan#9](https://github.com/nan-language/nan/issues/9).

## N7

**Receiver-charged transfer.**

- **Language 1:** a value moved out of a domain passes its chunks to the innermost domain enclosing both sides, still charged to the sender's cell (NaN 11 §Domain memory).
- **NaN OS requires:** the chunks of a moved value, and their charge, pass to the receiving domain and its cell.
- **Chapters:** [05](05-memory.md).
- **Issue:** [nan-language/nan#10](https://github.com/nan-language/nan/issues/10).

## N8

**Executor policy.**

- **Language 1:** the kernel executor is a fixed support routine with one global run queue and a periodic timer on vector `0x20` (NaN 27 §Descriptors, NaN ADR 0035).
- **NaN OS requires:** a seam through which the kernel supplies the executor's policy:
  - per-CPU run queues and wake IPIs;
  - per-entry accounting, classes and shares (`Executor.split`), and deadlines;
  - spawns held until a handler returns;
  - an idle hook, and a one-shot timer in place of the periodic tick;
  - the seeded serial executor outside tests.
- **Chapters:** [04](04-kernel.md), [05](05-memory.md), [06](06-execution.md), [08](08-scheduling.md), [09](09-authority.md), [16](16-network.md), [21](21-performance.md).
- **Issue:** [nan-language/nan#11](https://github.com/nan-language/nan/issues/11).

## N9

**Time in kernels.**

- **Language 1:** `boot::Kernel` yields no `sys::Clock`, and the kernel executor records none (NaN 24 §Anchor). Durations count milliseconds.
- **NaN OS requires:**
  - a kernel `sys::Clock` the executor records, so `task::within`, `Cx.after` and `sleep` work in kernels;
  - nanosecond time;
  - a wall-clock capability;
  - attenuation of a clock's resolution.
- **Chapters:** [02](02-identifiers.md), [06](06-execution.md), [08](08-scheduling.md), [09](09-authority.md).
- **Issue:** [nan-language/nan#12](https://github.com/nan-language/nan/issues/12).

## N10

**Inline capability checks.**

- **Language 1:** every operation through a Copy capability checks its generation in an out-of-line support routine. It takes 12.5–15 ns.
- **NaN OS requires:** lowering inlines the compare, and calls the routine only on a mismatch.
- **Chapters:** [21](21-performance.md).
- **Issue:** [nan-language/nan#13](https://github.com/nan-language/nan/issues/13).

## N11

**Intrinsics and cross-CPU operations.**

- **Language 1:** the privileged intrinsics are a closed list per descriptor (NaN 23 §Privileged intrinsics). `invalidate_page` is local, and kernel code can send no IPI.
- **NaN OS requires,** on x86_64:
  - `cpuid`, `rdtsc` and `rdtscp`, `monitor` and `mwait`, `hlt`;
  - `invpcid` and a PCID-tagged `write_cr3`;
  - `xgetbv` and `xsetbv`, `rdrand` and `rdseed`;
  - the CR4 features SMEP, SMAP, UMIP, PCIDE, FSGSBASE and TSD;
  - IBPB and VERW.

  On AArch64:
  - `wfi` and `wfe`;
  - TLBI by ASID;
  - cache maintenance;
  - `hvc` and `smc` for PSCI.

  On both architectures: IPIs and TLB shootdown.
- **Chapters:** [01](01-machines.md), [03](03-boot.md), [04](04-kernel.md), [09](09-authority.md).
- **Issue:** [nan-language/nan#14](https://github.com/nan-language/nan/issues/14).

## N12

**All memory mapped.**

- **Language 1:** NaN 24 §Physical memory requires the entry root to map all usable memory. The runtime maps only the first 4 GiB (NaN ADR 0035).
- **NaN OS requires:** an entry root mapping all usable memory, with 1 GiB pages where present and 2 MiB pages otherwise.
- **Chapters:** [04](04-kernel.md).
- **Issue:** [nan-language/nan#15](https://github.com/nan-language/nan/issues/15).

## N13

**IOMMU-modelled DMA.**

- **Language 1:** a DMA buffer's bus address is physical (NaN 23 §DMA), a register window needs a trusted premise (NaN 23 §Register windows), and every `device` grant is a trust escape.
- **NaN OS requires:**
  - an IOMMU domain capability, with IOVA bus addresses;
  - device cut-off before a driver's settlement frees DMA memory;
  - PCI function windows that need no trusted premise;
  - IOMMU-confined device grants that are not trust escapes.
- **Chapters:** [09](09-authority.md), [15](15-devices.md).
- **Issue:** [nan-language/nan#16](https://github.com/nan-language/nan/issues/16).

## N14

**Firmware services and hand-off.**

- **Language 1:** `boot::Services` offers `write_console` and `read_file`, and `exit_services` returns the `boot::Kernel` of the same image (NaN 24 §Firmware services). `boot::Kernel.regions` lists usable memory only, and `boot::Kernel.ring0` gives the only `hw::Ring0` (NaN 24 §Anchor).
- **NaN OS requires:**
  - **Firmware services:**
    - file writes, file deletion and directory listing on the boot volume;
    - the device path of the loader's image, giving its file and the boot volume's partition number and GPT unique partition GUID or MBR disk signature;
    - variables, including boot-services-only ones;
    - TCG2 measurement and the event log;
    - TPM 2.0 commands through `EFI_TCG2_PROTOCOL.SubmitCommand`, for NV indices;
    - EFI_RNG;
    - the configuration tables (RSDP, SMBIOS, device tree);
    - the memory map with the type of each range, before `exit_services` and as `exit_services` leaves it;
    - console input and a timer.
  - **Privileged intrinsics:** a `hw::Ring0` before `exit_services`.
  - **Hand-off:** entering a loaded bare image through its target's boot protocol.
  - **Kernel accessors:** `boot::Kernel` accessors for the start information's modules and firmware tables.
- **Chapters:** [01](01-machines.md), [03](03-boot.md), [04](04-kernel.md).
- **Issue:** [nan-language/nan#17](https://github.com/nan-language/nan/issues/17).

## N15

**Signatures.**

- **Language 1:** seals are unsigned and the author rule reads git (NaN 31 §Seals). The artefact signature, certificate and provenance sections are reserved (NaN 30 §Reserved sections).
- **NaN OS requires:**
  - Ed25519 signatures in seals;
  - an author rule over the author principals that the provenance names;
  - the signature and provenance sections in fixed formats a device verifies without the toolchain.

  Check certificates and lowered images belong to NaN OS's bundle format ([10](10-code.md)).
- **Chapters:** [04](04-kernel.md), [09](09-authority.md), [10](10-code.md), [14](14-changes.md), [18](18-link.md).
- **Issue:** [nan-language/nan#18](https://github.com/nan-language/nan/issues/18).

## N16

**Reversibility.**

- **Language 1:** the digest's eleventh field, reversibility, is reserved and never computed (NaN 10 §Reserved field), and limit words are a closed list.
- **NaN OS requires:**
  - the field computed by propagation over `none < recorded < restorable < irreversible{kinds}`, with kinds among `emit`, `actuate`, `erase` and `spend`;
  - a per-target class for each std primitive;
  - limit words that bound the field;
  - sealed classes for device operations, and sealed compensation claims.
- **Chapters:** [10](10-code.md), [14](14-changes.md).
- **Issue:** [nan-language/nan#19](https://github.com/nan-language/nan/issues/19).

## N17

**SIMD and codegen levels.**

- **Language 1:** `core::simd` is reserved, and lowering uses the base instruction set only (NaN 27 §Descriptors).
- **NaN OS requires:**
  - `core::simd` with fixed-width lanes and reductions in a fixed order;
  - codegen levels selectable at lowering.
- **Chapters:** [10](10-code.md), [21](21-performance.md).
- **Issue:** [nan-language/nan#20](https://github.com/nan-language/nan/issues/20).

## N19

**Construction-literal decoding.**

- **Language 1:** std decodes `mem::wire` and `text::json` (NaN 13 §Std encodings). Values written as NaN construction literals, the form source code and derived `Render` use, are decoded by nothing.
- **NaN OS requires:** a `core::Decoder` for NaN construction literals of any `Encode` type, with the validation of other decoders, predicates included.
- **Chapters:** [18](18-link.md).
- **Issue:** [nan-language/nan#21](https://github.com/nan-language/nan/issues/21).

## N20

**Schema reflection.**

- **Language 1:** code cannot read schema hashes (NaN 13 §Schema hashes), and code roots record component state schemas only (NaN 30 §Code roots).
- **NaN OS requires:**
  - a const schema hash and schema tree for each `Encode` type;
  - schema trees, with their type names, in code roots;
  - type-erased decoders that bundles export, checking predicates.
- **Chapters:** [02](02-identifiers.md), [07](07-communication.md), [10](10-code.md), [11](11-store.md), [14](14-changes.md), [17](17-observability.md), [18](18-link.md), [19](19-oversight.md), [20](20-base.md).
- **Issue:** [nan-language/nan#22](https://github.com/nan-language/nan/issues/22).

## N21

**Interrupt vectors.**

- **Language 1:** `hw::IrqShared.install` binds a vector on every CPU and traps on a vector already bound (NaN 24 §Locks). On AArch64, every interrupt from 32 up goes to the boot CPU (NaN 24 §Interrupt functions).
- **NaN OS requires:**
  - vectors bound per CPU;
  - several devices installed on one vector, all their interrupt fns running before the interrupt completes, for shared level-triggered lines;
  - x2APIC mode;
  - per-interrupt affinity and ITS MSIs on AArch64.
- **Chapters:** [04](04-kernel.md), [15](15-devices.md).
- **Issue:** [nan-language/nan#23](https://github.com/nan-language/nan/issues/23).

## N22

**The emulator and `nan run`.**

- **Language 1:** NaN 33 §Commands specifies only the serial port's output. The descriptor's machine has a FAT boot volume as its only disk, no NIC, IOMMU, TPM, watchdog or second serial port, no reboot, and QEMU's default CPU model (NaN 27 §Descriptors).
- **NaN OS requires:**
  - serial input from standard input, specified;
  - a persistent block device, a NIC, an IOMMU, a TPM, a watchdog and a second serial port;
  - reboot within a run;
  - a CPU model with the machine conditions of [01](01-machines.md).
- **Chapters:** [01](01-machines.md), [18](18-link.md).
- **Issue:** [nan-language/nan#24](https://github.com/nan-language/nan/issues/24).

## N23

**User mode.**

- **Language 1:** no operation enters user mode. The user-exception and system-call seams are checked but never entered (NaN 24 §User exceptions and syscalls).
- **NaN OS requires:**
  - entering ring 3 or EL0 on a root with a register state;
  - live user seams whose landing fns choose the next context;
  - user roots exempt from `write_cr3`'s mapping rule;
  - `UserAddr.copy_out` and bounded byte copies;
  - frames shared with user code, read as Untrusted;
  - a user-mode runtime.
- **Chapters:** [01](01-machines.md), [04](04-kernel.md), [05](05-memory.md), [06](06-execution.md), [07](07-communication.md), [09](09-authority.md), [11](11-store.md).
- **Issue:** [nan-language/nan#25](https://github.com/nan-language/nan/issues/25).

## N24

**Extended state on demand.**

- **Language 1:** every seam and `hw::switch_to` saves the complete extended state (NaN 24 §Floating point).
- **NaN OS requires:**
  - integer code lowered without vector registers;
  - extended state saved and restored only for threads that own it.
- **Chapters:** [21](21-performance.md).
- **Issue:** [nan-language/nan#26](https://github.com/nan-language/nan/issues/26).

## N25

**Constant-time code.**

- **Language 1:** no property bounds the timing or the memory accesses of code on secret values.
- **NaN OS requires:** a checked way to demand that a function's branches and memory accesses never depend on given inputs, honoured by lowering.
- **Chapters:** [09](09-authority.md), [16](16-network.md), [19](19-oversight.md).
- **Issue:** [nan-language/nan#27](https://github.com/nan-language/nan/issues/27).

## N26

**Verify mode.**

- **Language 1:** reused units keep their recorded statuses (NaN 30 §Reuse), and a proven status removes a run-time check.
- **NaN OS requires:** a mode that checks well-formedness, decides every obligation anew and runs every example, whatever the requires-set.
- **Chapters:** [10](10-code.md), [20](20-base.md).
- **Issue:** [nan-language/nan#28](https://github.com/nan-language/nan/issues/28).

## N27

**Kernel-implemented capability types.**

- **Language 1:** primitive capabilities are std types only (NaN 05 §Capability types). Implementing `sys::Capability` elsewhere is `authority.capability-impl`.
- **NaN OS requires:** capability types whose operations dispatch to kernel functions. They are revocable and attenuable, appear in digests and authority summaries, and can be replaced by recording substitutes outside tests.
- **Chapters:** [04](04-kernel.md), [05](05-memory.md), [06](06-execution.md), [07](07-communication.md), [09](09-authority.md), [10](10-code.md), [11](11-store.md), [14](14-changes.md), [15](15-devices.md), [16](16-network.md).
- **Issue:** [nan-language/nan#29](https://github.com/nan-language/nan/issues/29).

## N28

**Linear resources of components.**

- **Language 1:** a component's Env and messages cannot be linear (NaN 26 §Components).
- **NaN OS requires:** either a linear Env the runner consumes through abandon methods, or runner-held linear resources that a handler reaches through Inert ids.
- **Chapters:** [06](06-execution.md), [09](09-authority.md), [15](15-devices.md).
- **Issue:** [nan-language/nan#30](https://github.com/nan-language/nan/issues/30).

## N29

**Encoded Untrusted fields.**

- **Language 1:** `Untrusted[T]` has no `Encode` (NaN 13 §Std encodings).
- **NaN OS requires:** `Untrusted[T]: Encode`, decoded structurally with the result opaque, so derived decoders stay parsing.
- **Chapters:** [07](07-communication.md), [11](11-store.md), [15](15-devices.md), [18](18-link.md), [19](19-oversight.md).
- **Issue:** [nan-language/nan#31](https://github.com/nan-language/nan/issues/31).

## N30

**AArch64 machines.**

- **Language 1:** v0.1.0 has no AArch64 targets. At NaN's main branch, the AArch64 machine descriptors fix QEMU `virt`'s memory base, UART, GIC and PSCI conduit (NaN 27 §Descriptors).
- **NaN OS requires:** AArch64 machine targets that discover these from the device tree or ACPI.
- **Chapters:** [01](01-machines.md), [03](03-boot.md).
- **Issue:** [nan-language/nan#32](https://github.com/nan-language/nan/issues/32).

## N31

**Kernel placement.**

- **Language 1:** an `x86_64-bare` image loads at fixed physical addresses that overlap OVMF's reserved memory.
- **NaN OS requires:** a loader can place a bare kernel in usable memory, whether by relocatable images or by a placement clear of firmware memory.
- **Chapters:** [03](03-boot.md).
- **Issue:** [nan-language/nan#2](https://github.com/nan-language/nan/issues/2).
