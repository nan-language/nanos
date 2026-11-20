# 0119. Stack on NaN v0.22.0

Status: superseded by 0132

## Context

NaN OS is an operating system built for AI agents from the root. Agents write it, so its language must be one they write and read reliably, and every change must be observable without real hardware. It must also work on the machines agents run on, many of which are AArch64.

Release v0.22.0 lets a kernel grow its boot heap by a region through `boot::Kernel.grow_heap`, from which the runtime draws its records, task stacks, bundle placements and cell chunks once the boot area's heap is full; records the protocol schema of each component entry of a bundle and gives it through `sys::load::Load.schema`; gives a missing field of an open type the empty container for strings, byte strings, lists, sets and maps, and `None` for an `Option`; puts the images of the nanos target of a UEFI image's architecture on its boot volume under `nan run`; and reports a dependency outside the workspace on its `dep` directive. It keeps what v0.21.0 gave: a kernel installing its own roots runs the bundles it loads; text formats decode open types and types holding themselves within a stack bound; a preempted kernel task kept off another CPU's records; one workspace holding the programs NaN OS runs beside its kernel, `nan build` building each nanos program as a bundle against the interface file of the one kernel of its architecture that the same build writes, and a nanos target's tests running as bundles, each alone in an isolated space; the artefacts of a bundle's final closure written beside it; a kernel reading its interface hash and what its boot heap holds; a holder encoding every message its component handles, whoever posted it; the NaN toolchain built for the nanos targets, running as a job in an isolated space whose world reaches files through system calls; and a `sys::Fs` that lists, creates and renames directories. The boot area's heap stays 16 MiB, and an allocation above 64 KiB takes whole 64 KiB chunks.

## Decision

- NaN, language version 1, with its toolchain `nan` at release v0.22.0, pinned by `tools/install`.
- The loader moves the kernel by the least multiple of 2 MiB that puts all its segments in one range of free memory, and the kernel maps its code where `boot::Kernel.image_segments` says its segments lie.
- The toolchain is the only build tool: `nan check`, `nan fmt`, `nan test`, `nan build` and `nan run`.
- The repository root is one NaN workspace, each component a package: `loader`, the boot loader, for `x86_64-uefi`, `kernel` for `x86_64-bare`, and the libraries they use for `any`, whose tests run on the host.
- QEMU with OVMF and swtpm, as NaN's target descriptors fix them, runs every image and every test of the machine targets, with KVM where the host offers it.
- The specification in `docs/spec/` defines NaN OS for x86_64 and AArch64. Architecture-specific code lives in NaN target variants (`name@target`), so the rest is shared. The x86_64 machine is the one the sandbox runs.
- A task's stack bound is a round size just above the use its final build verifies.
- An incarnation's account runs on its cell, so the tasks of its domain draw their stacks through the cell and the boot heap holds only the core's tasks and the runner's.
- At start step 3 the kernel grows the boot heap by RAM ÷ 32, set aside from the top of the richest node pool's region below the part it leaves to the frame allocator, and the memory summary counts it in `boot_heap`.

Alternatives rejected:

- **Staying on v0.21.0:** the 16 MiB boot heap held 14.2 MiB once the base instances ran, so an instance of a loaded bundle and a job could not run at once (nan-language/nan#254), and a kernel could not describe the protocol of a component it held from a bundle (nan-language/nan#255).

## Consequences

- Agents learn NaN from `nan primer` rather than from training.
- The machine is the one the descriptors fix: q35 with an IOMMU, 4 processors, 512 MiB, an entropy device, a TPM, a watchdog, a virtio disk, an NVMe namespace, a SATA disk, an empty hot-plug slot, a NIC, a serial and a console port and a monitor socket, no display. Changing it means changing NaN.
- A host needs Linux on x86_64, a C compiler as `cc` for the host tests, QEMU 10.2 or later, OVMF and swtpm.
- A change that makes a task use more stack than its bound fails the final build with `limit.stack-exceeded`, and raises the bound then.
- An instance's memory limit covers the stacks of its domain's tasks.
- Every defect or gap found in NaN goes to its repository rather than into a workaround; `docs/spec/22-nan.md` lists the features NaN OS requires, each with its issue.
