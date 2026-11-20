# 0132. Stack on NaN v0.23.0

Status: superseded by 0137

## Context

NaN OS is an operating system built for AI agents from the root. Agents write it, so its language must be one they write and read reliably, and every change must be observable without real hardware. It must also work on the machines agents run on, many of which are AArch64.

Release v0.23.0 lets a kernel holding the component of a nanos bundle take the messages it sends itself and list, start and drop its held keyed tasks, and step it through `Instance.step_encoded`, which gives the encoding of the call each step handles; passes a component's checkpoint with its schema hash as `nanos::Checkpoint`, which `Instance.checkpoint` gives and `hold` decodes through the migrations of the state; records in a code root the schema hash of each frozen shape a type's forward migrations convert from and whether an inverse converts back; lets `decode_from` read an earlier schema the history records whose tree the type's accepts; checks a struct's predicates at every construction; and writes the artefacts of a machine image's final closure beside it. It keeps what v0.22.0 gave: a boot heap a kernel grows by a region through `boot::Kernel.grow_heap`, from which the runtime draws its records, task stacks, bundle placements and cell chunks once the boot area's heap is full; the protocol schema of each component entry of a bundle, which `sys::load::Load.schema` gives; the empty container, or `None` for an `Option`, for a missing field of an open type; the images of the nanos target of a UEFI image's architecture on its boot volume under `nan run`; and a dependency outside the workspace reported on its `dep` directive; and what earlier releases gave: a kernel installing its own roots runs the bundles it loads; text formats decode open types and types holding themselves within a stack bound; a preempted kernel task kept off another CPU's records; one workspace holding the programs NaN OS runs beside its kernel, `nan build` building each nanos program as a bundle against the interface file of the one kernel of its architecture that the same build writes, and a nanos target's tests running as bundles, each alone in an isolated space; the artefacts of a bundle's final closure written beside it; a kernel reading its interface hash and what its boot heap holds; a holder encoding every message its component handles, whoever posted it; the NaN toolchain built for the nanos targets, running as a job in an isolated space whose world reaches files through system calls; and a `sys::Fs` that lists, creates and renames directories. The boot area's heap stays 16 MiB, and an allocation above 64 KiB takes whole 64 KiB chunks.

## Decision

- NaN, language version 1, with its toolchain `nan` at release v0.23.0, pinned by `tools/install`.
- The loader moves the kernel by the least multiple of 2 MiB that puts all its segments in one range of free memory, and the kernel maps its code where `boot::Kernel.image_segments` says its segments lie.
- The toolchain is the only build tool: `nan check`, `nan fmt`, `nan test`, `nan build` and `nan run`.
- The repository root is one NaN workspace, each component a package: `loader`, the boot loader, for `x86_64-uefi`, `kernel` for `x86_64-bare`, and the libraries they use for `any`, whose tests run on the host.
- QEMU with OVMF and swtpm, as NaN's target descriptors fix them, runs every image and every test of the machine targets, with KVM where the host offers it.
- The specification in `docs/spec/` defines NaN OS for x86_64 and AArch64. Architecture-specific code lives in NaN target variants (`name@target`), so the rest is shared. The x86_64 machine is the one the sandbox runs.
- A task's stack bound is a round size just above the use its final build verifies.
- An incarnation's account runs on its cell, so the tasks of its domain draw their stacks through the cell and the boot heap holds only the core's tasks and the runner's.
- At start step 3 the kernel grows the boot heap by RAM ÷ 32, set aside from the top of the richest node pool's region below the part it leaves to the frame allocator, and the memory summary counts it in `boot_heap`.

Alternatives rejected:

- **Staying on v0.22.0:** no artefact told which older schemas a `migrate` covers (nan-language/nan#261), `decode_from` refused an earlier schema its type's tree accepts (nan-language/nan#262), and a kernel could neither hold the effects of a bundle component's step nor see the message it handled (nan-language/nan#263, nan-language/nan#264), so no update could rely on a `migrate`, and bundle components could be neither recorded nor replayed on a revert.

## Consequences

- Agents learn NaN from `nan primer` rather than from training.
- The machine is the one the descriptors fix: q35 with an IOMMU, 4 processors, 512 MiB, an entropy device, a TPM, a watchdog, a virtio disk, an NVMe namespace, a SATA disk, an empty hot-plug slot, a NIC, a serial and a console port and a monitor socket, no display. Changing it means changing NaN.
- A host needs Linux on x86_64, a C compiler as `cc` for the host tests, QEMU 10.2 or later, OVMF and swtpm.
- A change that makes a task use more stack than its bound fails the final build with `limit.stack-exceeded`, and raises the bound then.
- An instance's memory limit covers the stacks of its domain's tasks.
- Every defect or gap found in NaN goes to its repository rather than into a workaround; `docs/spec/22-nan.md` lists the features NaN OS requires, each with its issue.
