# 0003. Stack and architectures

Status: accepted

## Context

NaN OS is an operating system built for AI agents from the root. Agents write it, so its language must be one they write and read reliably, and every change must be observable without real hardware. It must also work on the machines agents run on, many of which are AArch64; NaN added AArch64 targets after its release v0.1.0, which has none.

## Decision

- NaN, language version 1, with its toolchain `nan` at release v0.1.0, pinned by `tools/install`.
- The toolchain is the only build tool: `nan check`, `nan fmt`, `nan test`, `nan build` and `nan run`.
- The repository root is one NaN workspace, each component a package: `loader`, the boot loader, for `x86_64-uefi`, and `kernel` for `x86_64-bare`.
- QEMU with OVMF, as NaN's target descriptors fix them, runs every image and every test of those targets, with KVM where the host offers it.
- The specification in `docs/spec/` defines NaN OS for x86_64 and AArch64. Architecture-specific code lives in NaN target variants (`name@target`), so the rest is shared. The x86_64 machine is the one the sandbox runs.

Alternatives rejected:

- **x86_64 only:** much of the hardware agents run on would be out of reach.
- **A specification per architecture:** the two would drift, and most rules do not depend on the CPU.

## Consequences

- Agents learn NaN from `nan primer` rather than from training.
- Building for AArch64 needs a pinned NaN release with AArch64 targets.
- The machine is the one the descriptors fix: q35, 4 processors, 512 MiB, an entropy device, a serial console, no display and no network. Changing it means changing NaN.
- A host needs Linux on x86_64, QEMU 10.2 or later and OVMF.
- Every defect or gap found in NaN goes to its repository rather than into a workaround; `docs/spec/22-nan.md` lists the features NaN OS requires, each with its issue.
