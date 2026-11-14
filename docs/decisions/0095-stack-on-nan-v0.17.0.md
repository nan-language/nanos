# 0095. Stack on NaN v0.17.0

Status: accepted

## Context

NaN OS is an operating system built for AI agents from the root. Agents write it, so its language must be one they write and read reliably, and every change must be observable without real hardware. It must also work on the machines agents run on, many of which are AArch64.

Release v0.17.0 gives what chapters 05, 06 and 08 wait for: a watch of a cell's charge against thresholds, an account's executor number and a share changing while its tasks run, a deadline ordering scheduling without cancelling and the calling task's deadline, a stepper's count of its keyed tasks, and whether a shared value has another handle. It probes a model-specific register without faulting, and its emulator halts a processor in MWAIT. Its host targets are named after their operating system, `x86_64-linux` among them. It keeps what v0.16.0 gave. The boot heap stays 16 MiB, and an allocation above 64 KiB takes whole 64 KiB chunks.

## Decision

- NaN, language version 1, with its toolchain `nan` at release v0.17.0, pinned by `tools/install`.
- The loader moves the kernel by the least multiple of 2 MiB that puts all its segments in one range of free memory, and the kernel maps its code where `boot::Kernel.image_segments` says its segments lie.
- The toolchain is the only build tool: `nan check`, `nan fmt`, `nan test`, `nan build` and `nan run`.
- The repository root is one NaN workspace, each component a package: `loader`, the boot loader, for `x86_64-uefi`, `kernel` for `x86_64-bare`, and the libraries they use for `any`, whose tests run on the host.
- QEMU with OVMF and swtpm, as NaN's target descriptors fix them, runs every image and every test of the machine targets, with KVM where the host offers it.
- The specification in `docs/spec/` defines NaN OS for x86_64 and AArch64. Architecture-specific code lives in NaN target variants (`name@target`), so the rest is shared. The x86_64 machine is the one the sandbox runs.
- A task's stack bound is a round size just above the use its final build verifies.
- An incarnation's account runs on its cell, so the tasks of its domain draw their stacks through the cell and the boot heap holds only the core's tasks and the runner's.

Alternatives rejected:

- **Staying on v0.16.0:** pool levels would follow draws only when the memory keeper is asked, an account's share would stay fixed for its life, the policy could keep no state per account and see no deadline that cancels nothing, an instance delaying a message would never be idle, the store could not tell a cached piece some reader holds, idle processors would halt rather than enter MWAIT, and RAPL would be read only on the processor models a table lists (nan-language/nan#197, #198, #199, #200, #201, #202, #220 and #222).

## Consequences

- Agents learn NaN from `nan primer` rather than from training.
- The machine is the one the descriptors fix: q35 with an IOMMU, 4 processors, 512 MiB, an entropy device, a TPM, a watchdog, a virtio disk, an NVMe namespace, a SATA disk, an empty hot-plug slot, a NIC, a serial and a console port and a monitor socket, no display. Changing it means changing NaN.
- A host needs Linux on x86_64, a C compiler as `cc` for the host tests, QEMU 10.2 or later, OVMF and swtpm.
- A change that makes a task use more stack than its bound fails the final build with `limit.stack-exceeded`, and raises the bound then.
- An instance's memory limit covers the stacks of its domain's tasks.
- Every defect or gap found in NaN goes to its repository rather than into a workaround; `docs/spec/22-nan.md` lists the features NaN OS requires, each with its issue.
