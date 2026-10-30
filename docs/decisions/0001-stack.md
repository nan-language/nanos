# 0001. Stack

Status: superseded by 0003

## Context

NaN OS is an operating system built for AI agents from the root, from the boot loader and kernel to drivers and user interfaces, rather than AI added on top of a system made for humans. Agents write it, so its language must be one they write and read reliably. It targets x86_64 only for now, and every change must be observable without real hardware.

## Decision

- NaN, language version 1, the language made for agentic coding, with its toolchain `nan` at release v0.1.0, pinned by `tools/install`.
- The toolchain is the only build tool: `nan check` reports diagnostics, `nan fmt` formats, `nan test` runs the tests of a package, module or item, `nan build` and `nan run` build and boot images.
- The repository root is one NaN workspace, each component a package: `loader`, the boot loader, for `x86_64-uefi`, and `kernel` for `x86_64-bare`.
- QEMU with OVMF, as NaN's target descriptors fix them, runs every image and every test of those targets, with KVM where the host offers it.

## Consequences

- Agents learn NaN from `nan primer` rather than from training.
- Language 1 leaves out what only NaN OS consumes, such as IPC, program loading, live update, a UI model and signed seals, and reserves the `x86_64-nanos` target. NaN OS needs them added to NaN, and every defect or gap found in NaN goes to its repository rather than into a workaround in another language.
- The machine is the one the descriptors fix: q35, 4 processors, 512 MiB, an entropy device, a serial console, no display and no network. Changing it means changing NaN.
- A host needs Linux on x86_64, QEMU 10.2 or later and OVMF.
