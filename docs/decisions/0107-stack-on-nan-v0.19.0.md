# 0107. Stack on NaN v0.19.0

Status: superseded by 0109

## Context

NaN OS is an operating system built for AI agents from the root. Agents write it, so its language must be one they write and read reliably, and every change must be observable without real hardware. It must also work on the machines agents run on, many of which are AArch64.

Releases v0.18.0 and v0.19.0 give what chapters 10 and 17 wait for: a bundle build writes the artefacts of its final closure beside the bundle, a kernel reads its interface hash and what its boot heap holds, a holder encodes every message its component handles, whoever posted it, and the NaN toolchain builds for the nanos targets, running as a job in an isolated space whose world reaches files through system calls. A `sys::Fs` also lists, creates and renames directories. They add the macOS host targets and keep what v0.17.0 gave. The boot heap stays 16 MiB, and an allocation above 64 KiB takes whole 64 KiB chunks.

## Decision

- NaN, language version 1, with its toolchain `nan` at release v0.19.0, pinned by `tools/install`.
- The loader moves the kernel by the least multiple of 2 MiB that puts all its segments in one range of free memory, and the kernel maps its code where `boot::Kernel.image_segments` says its segments lie.
- The toolchain is the only build tool: `nan check`, `nan fmt`, `nan test`, `nan build` and `nan run`.
- The repository root is one NaN workspace, each component a package: `loader`, the boot loader, for `x86_64-uefi`, `kernel` for `x86_64-bare`, and the libraries they use for `any`, whose tests run on the host.
- QEMU with OVMF and swtpm, as NaN's target descriptors fix them, runs every image and every test of the machine targets, with KVM where the host offers it.
- The specification in `docs/spec/` defines NaN OS for x86_64 and AArch64. Architecture-specific code lives in NaN target variants (`name@target`), so the rest is shared. The x86_64 machine is the one the sandbox runs.
- A task's stack bound is a round size just above the use its final build verifies.
- An incarnation's account runs on its cell, so the tasks of its domain draw their stacks through the cell and the boot heap holds only the core's tasks and the runner's.

Alternatives rejected:

- **Staying on v0.17.0:** NaN OS could make no bundle file, tell no image built for its kernel from another, record no message a keyed task posts, run no toolchain on the machine, and not read what fills its boot heap (nan-language/nan#225, #229, #230, #231 and #232).

## Consequences

- Agents learn NaN from `nan primer` rather than from training.
- The machine is the one the descriptors fix: q35 with an IOMMU, 4 processors, 512 MiB, an entropy device, a TPM, a watchdog, a virtio disk, an NVMe namespace, a SATA disk, an empty hot-plug slot, a NIC, a serial and a console port and a monitor socket, no display. Changing it means changing NaN.
- A host needs Linux on x86_64, a C compiler as `cc` for the host tests, QEMU 10.2 or later, OVMF and swtpm.
- A change that makes a task use more stack than its bound fails the final build with `limit.stack-exceeded`, and raises the bound then.
- An instance's memory limit covers the stacks of its domain's tasks.
- Every defect or gap found in NaN goes to its repository rather than into a workaround; `docs/spec/22-nan.md` lists the features NaN OS requires, each with its issue.
