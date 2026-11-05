# 0016. Stack on NaN v0.4.0

Status: accepted

## Context

NaN OS is an operating system built for AI agents from the root. Agents write it, so its language must be one they write and read reliably, and every change must be observable without real hardware. It must also work on the machines agents run on, many of which are AArch64.

Release v0.4.0 changes the kernel runtime the core is built on: a policy places a task bound to its CPU and steals several tasks at once, a timer deadline takes the slack a policy gives it, an idle CPU may wait in `mwait`, the kernel clock runs from the HPET or the ACPI PM timer where the time-stamp counter is not invariant, every x86_64 image enables no-execute pages and kernel write protection, `boot::Kernel` gives the memory map, the image's segments and the boot area, and `hw::Region.split` divides a region.

## Decision

- NaN, language version 1, with its toolchain `nan` at release v0.4.0, pinned by `tools/install`.
- The toolchain is the only build tool: `nan check`, `nan fmt`, `nan test`, `nan build` and `nan run`.
- The repository root is one NaN workspace, each component a package: `loader`, the boot loader, for `x86_64-uefi`, `kernel` for `x86_64-bare`, and the libraries they use for `any`, whose tests run on the host.
- QEMU with OVMF and swtpm, as NaN's target descriptors fix them, runs every image and every test of the machine targets, with KVM where the host offers it.
- The specification in `docs/spec/` defines NaN OS for x86_64 and AArch64. Architecture-specific code lives in NaN target variants (`name@target`), so the rest is shared. The x86_64 machine is the one the sandbox runs.

Alternatives rejected:

- **Staying on v0.3.1:** its policy interface no longer matches the runtime the core needs, and the gaps that ADR 0015 works around stay open.
- **x86_64 only:** much of the hardware agents run on would be out of reach.
- **A specification per architecture:** the two would drift, and most rules do not depend on the CPU.

## Consequences

- Agents learn NaN from `nan primer` rather than from training.
- The machine is the one the descriptors fix: q35 with an IOMMU, 4 processors, 512 MiB, an entropy device, a TPM, a watchdog, a disk, a NIC, a serial and a console port, no display. Changing it means changing NaN.
- A host needs Linux on x86_64, a C compiler as `cc` for the host tests, QEMU 10.2 or later, OVMF and swtpm.
- Every defect or gap found in NaN goes to its repository rather than into a workaround; `docs/spec/22-nan.md` lists the features NaN OS requires, each with its issue.
