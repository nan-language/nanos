# 0046. Stack on NaN v0.10.0

Status: superseded by 0049

## Context

NaN OS is an operating system built for AI agents from the root. Agents write it, so its language must be one they write and read reliably, and every change must be observable without real hardware. It must also work on the machines agents run on, many of which are AArch64.

Release v0.9.0 shares the stack slots of values never live at once, refuses a `var` bound to a writable part of a Copy type, lets `nan dep sync` resolve an approved final-link root, fails decoders with the path of the field they rejected, and lets a bare kernel clear a boot module and set a PCI link's power management. Release v0.10.0 lets a capability kind's record be of any fixed size, so a kind attenuates by a path; lets a handle record, a struct of encodable and capability fields, be an operation's type; keeps each cell's highest charge, read through `Budget.peak`; gives erased decoders a stack machine-target code can call them on; and keeps the moved part of a var assigned again.

## Decision

- NaN, language version 1, with its toolchain `nan` at release v0.10.0, pinned by `tools/install`.
- The toolchain is the only build tool: `nan check`, `nan fmt`, `nan test`, `nan build` and `nan run`.
- The repository root is one NaN workspace, each component a package: `loader`, the boot loader, for `x86_64-uefi`, `kernel` for `x86_64-bare`, and the libraries they use for `any`, whose tests run on the host.
- QEMU with OVMF and swtpm, as NaN's target descriptors fix them, runs every image and every test of the machine targets, with KVM where the host offers it.
- The specification in `docs/spec/` defines NaN OS for x86_64 and AArch64. Architecture-specific code lives in NaN target variants (`name@target`), so the rest is shared. The x86_64 machine is the one the sandbox runs.

Alternatives rejected:

- **Staying on v0.9.0:** a View could not narrow to a subtree, `branch` could not give its record beside its View, and an instance's or account's memory peak could not be read.

## Consequences

- Agents learn NaN from `nan primer` rather than from training.
- The machine is the one the descriptors fix: q35 with an IOMMU, 4 processors, 512 MiB, an entropy device, a TPM, a watchdog, a virtio disk, an NVMe namespace, a SATA disk, an empty hot-plug slot, a NIC, a serial and a console port and a monitor socket, no display. Changing it means changing NaN.
- A host needs Linux on x86_64, a C compiler as `cc` for the host tests, QEMU 10.2 or later, OVMF and swtpm.
- The kernel's segments load at fixed physical addresses from 1 MiB, and the firmware keeps memory from 8 MiB, so the kernel image, zero-fill included, must stay below 7 MiB until NaN lets a loader place it higher ([nan-language/nan#146](https://github.com/nan-language/nan/issues/146)).
- Every defect or gap found in NaN goes to its repository rather than into a workaround; `docs/spec/22-nan.md` lists the features NaN OS requires, each with its issue.
