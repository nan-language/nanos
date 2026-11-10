# 0049. Stack on NaN v0.10.1

Status: accepted

## Context

NaN OS is an operating system built for AI agents from the root. Agents write it, so its language must be one they write and read reliably, and every change must be observable without real hardware. It must also work on the machines agents run on, many of which are AArch64.

Release v0.10.0 lets a capability kind's record be of any fixed size, so a kind attenuates by a path; lets a handle record be an operation's type; keeps each cell's highest charge, read through `Budget.peak`; gives erased decoders a stack machine-target code can call them on; and keeps the moved part of a var assigned again. Release v0.10.1 lets an `x86_64-bare` image move by a multiple of 2 MiB below 4 GiB, its entry seam adjusting the image's absolute addresses. The firmware keeps memory from 8 MiB, which a kernel loaded at 1 MiB reaches once its image passes 7 MiB.

## Decision

- NaN, language version 1, with its toolchain `nan` at release v0.10.1, pinned by `tools/install`.
- The loader moves the kernel by the least multiple of 2 MiB that puts all its segments in one range of free memory, and the kernel maps its code where `boot::Kernel.image_segments` says its segments lie.
- The toolchain is the only build tool: `nan check`, `nan fmt`, `nan test`, `nan build` and `nan run`.
- The repository root is one NaN workspace, each component a package: `loader`, the boot loader, for `x86_64-uefi`, `kernel` for `x86_64-bare`, and the libraries they use for `any`, whose tests run on the host.
- QEMU with OVMF and swtpm, as NaN's target descriptors fix them, runs every image and every test of the machine targets, with KVM where the host offers it.
- The specification in `docs/spec/` defines NaN OS for x86_64 and AArch64. Architecture-specific code lives in NaN target variants (`name@target`), so the rest is shared. The x86_64 machine is the one the sandbox runs.

Alternatives rejected:

- **Staying on v0.10.0:** the kernel could not grow past 7 MiB.

## Consequences

- Agents learn NaN from `nan primer` rather than from training.
- The machine is the one the descriptors fix: q35 with an IOMMU, 4 processors, 512 MiB, an entropy device, a TPM, a watchdog, a virtio disk, an NVMe namespace, a SATA disk, an empty hot-plug slot, a NIC, a serial and a console port and a monitor socket, no display. Changing it means changing NaN.
- A host needs Linux on x86_64, a C compiler as `cc` for the host tests, QEMU 10.2 or later, OVMF and swtpm.
- Every task of the kernel, instances' included, draws its stack and record from NaN's 16 MiB boot heap, about 60 KiB each whatever its bound, so the kernel runs fewer than 300 tasks at once ([nan-language/nan#147](https://github.com/nan-language/nan/issues/147)).
- Every defect or gap found in NaN goes to its repository rather than into a workaround; `docs/spec/22-nan.md` lists the features NaN OS requires, each with its issue.
