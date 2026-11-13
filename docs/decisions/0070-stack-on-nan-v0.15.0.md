# 0070. Stack on NaN v0.15.0

Status: superseded by 0074

## Context

NaN OS is an operating system built for AI agents from the root. Agents write it, so its language must be one they write and read reliably, and every change must be observable without real hardware. It must also work on the machines agents run on, many of which are AArch64.

Release v0.15.0 translates construction text through a named schema with work lists of its own, so kernels call it within their stack bounds; lists every operation of a protocol with its class and encodes and decodes a message of any operation, `pub` or not; lets a stepper's holder list its held keyed tasks with the class of the code each runs and start or cancel them one at a time; and lets another task wake a holder waiting for its intake. It keeps what v0.14.0 gave: construction text reading a type's own rendering through `core::Parse`, named type schemas, each operation's class and each parameter's position, the messages and held keyed tasks a holder takes without running them, served clocks, `text::base64`, and `sys::Stepper.step` returning what it handled. The boot heap stays 16 MiB, and an allocation above 64 KiB takes whole 64 KiB chunks.

## Decision

- NaN, language version 1, with its toolchain `nan` at release v0.15.0, pinned by `tools/install`.
- The loader moves the kernel by the least multiple of 2 MiB that puts all its segments in one range of free memory, and the kernel maps its code where `boot::Kernel.image_segments` says its segments lie.
- The toolchain is the only build tool: `nan check`, `nan fmt`, `nan test`, `nan build` and `nan run`.
- The repository root is one NaN workspace, each component a package: `loader`, the boot loader, for `x86_64-uefi`, `kernel` for `x86_64-bare`, and the libraries they use for `any`, whose tests run on the host.
- QEMU with OVMF and swtpm, as NaN's target descriptors fix them, runs every image and every test of the machine targets, with KVM where the host offers it.
- The specification in `docs/spec/` defines NaN OS for x86_64 and AArch64. Architecture-specific code lives in NaN target variants (`name@target`), so the rest is shared. The x86_64 machine is the one the sandbox runs.
- A task's stack bound is a round size just above the use its final build verifies.
- An incarnation's account runs on its cell, so the tasks of its domain draw their stacks through the cell and the boot heap holds only the core's tasks and the runner's.

Alternatives rejected:

- **Staying on v0.14.0:** the kernel could not translate a `call`'s arguments nor its reply, whose recursion no stack bound held; a holder could neither encode the messages its component sent itself through operations that are not `pub` nor list or release its held keyed tasks by class; and it could not be woken to fork its instance at quiescence or to pause its intake.

## Consequences

- Agents learn NaN from `nan primer` rather than from training.
- The machine is the one the descriptors fix: q35 with an IOMMU, 4 processors, 512 MiB, an entropy device, a TPM, a watchdog, a virtio disk, an NVMe namespace, a SATA disk, an empty hot-plug slot, a NIC, a serial and a console port and a monitor socket, no display. Changing it means changing NaN.
- A host needs Linux on x86_64, a C compiler as `cc` for the host tests, QEMU 10.2 or later, OVMF and swtpm.
- A change that makes a task use more stack than its bound fails the final build with `limit.stack-exceeded`, and raises the bound then.
- An instance's memory limit covers the stacks of its domain's tasks.
- Every defect or gap found in NaN goes to its repository rather than into a workaround; `docs/spec/22-nan.md` lists the features NaN OS requires, each with its issue.
