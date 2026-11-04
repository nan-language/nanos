# 0013. Stack on NaN v0.3.0

Status: accepted

## Context

NaN OS is an operating system built for AI agents from the root. Agents write it, so its language must be one they write and read reliably, and every change must be observable without real hardware. It must also work on the machines agents run on, many of which are AArch64.

Release v0.1.0 gives a UEFI program no firmware service beyond console output and file reads, and no way to enter a loaded kernel, and its bare images overlap OVMF's reserved memory: the loader cannot be built on it. Release v0.3.0 provides the features `docs/spec/22-nan.md` lists, firmware services, the hand-off and kernel placement among them, and its emulator machine adds an IOMMU, a TPM, a watchdog, a disk, a NIC, a console port and reboot within a run.

## Decision

- NaN, language version 1, with its toolchain `nan` at release v0.3.0, pinned by `tools/install`.
- The toolchain is the only build tool: `nan check`, `nan fmt`, `nan test`, `nan build` and `nan run`.
- The repository root is one NaN workspace, each component a package: `loader`, the boot loader, for `x86_64-uefi`, `kernel` for `x86_64-bare`, and the libraries they share for `any`, whose tests run on the host.
- QEMU with OVMF and swtpm, as NaN's target descriptors fix them, runs every image and every test of the machine targets, with KVM where the host offers it.
- The specification in `docs/spec/` defines NaN OS for x86_64 and AArch64. Architecture-specific code lives in NaN target variants (`name@target`), so the rest is shared. The x86_64 machine is the one the sandbox runs.

Alternatives rejected:

- **Staying on v0.1.0:** the loader could neither write the boot record, draw entropy nor enter the kernel.
- **x86_64 only:** much of the hardware agents run on would be out of reach.
- **A specification per architecture:** the two would drift, and most rules do not depend on the CPU.

## Consequences

- Agents learn NaN from `nan primer` rather than from training.
- The machine is the one the descriptors fix: q35 with an IOMMU, 4 processors, 512 MiB, an entropy device, a TPM, a watchdog, a disk, a NIC, a serial and a console port, no display. Changing it means changing NaN.
- A host needs Linux on x86_64, a C compiler as `cc` for the host tests, QEMU 10.2 or later, OVMF and swtpm.
- Every defect or gap found in NaN goes to its repository rather than into a workaround; `docs/spec/22-nan.md` lists the features NaN OS requires, each with its issue.
