# 0056. Stack on NaN v0.11.0

Status: accepted

## Context

NaN OS is an operating system built for AI agents from the root. Agents write it, so its language must be one they write and read reliably, and every change must be observable without real hardware. It must also work on the machines agents run on, many of which are AArch64.

Release v0.11.0 gives each task a header that the tasks it starts and the messages it queues carry, delivered by `Stepper.deliver_headed` and read through `sys::current_header`; hands a value to a channel without waiting through `task::Sender.offer`; and shrinks what a kernel task takes of the boot heap to its stack bound plus the image's routine reserve, about 6 KiB in NaN OS's kernel, with its record. The boot heap stays 16 MiB, and an allocation above 64 KiB takes whole 64 KiB chunks, so a task bounded to 64 KiB takes 128 KiB. Every task of the kernel, the runner's for each instance and each instance's keyed tasks included, draws from the boot heap.

## Decision

- NaN, language version 1, with its toolchain `nan` at release v0.11.0, pinned by `tools/install`.
- The loader moves the kernel by the least multiple of 2 MiB that puts all its segments in one range of free memory, and the kernel maps its code where `boot::Kernel.image_segments` says its segments lie.
- The toolchain is the only build tool: `nan check`, `nan fmt`, `nan test`, `nan build` and `nan run`.
- The repository root is one NaN workspace, each component a package: `loader`, the boot loader, for `x86_64-uefi`, `kernel` for `x86_64-bare`, and the libraries they use for `any`, whose tests run on the host.
- QEMU with OVMF and swtpm, as NaN's target descriptors fix them, runs every image and every test of the machine targets, with KVM where the host offers it.
- The specification in `docs/spec/` defines NaN OS for x86_64 and AArch64. Architecture-specific code lives in NaN target variants (`name@target`), so the rest is shared. The x86_64 machine is the one the sandbox runs.
- A task's stack bound is a round size just above the use its final build verifies, so the boot heap holds the tasks of every base instance and of the devices they run.

Alternatives rejected:

- **Staying on v0.10.1:** a kernel task took about 60 KiB of the boot heap whatever its bound, and the base instances did not fit.

## Consequences

- Agents learn NaN from `nan primer` rather than from training.
- The machine is the one the descriptors fix: q35 with an IOMMU, 4 processors, 512 MiB, an entropy device, a TPM, a watchdog, a virtio disk, an NVMe namespace, a SATA disk, an empty hot-plug slot, a NIC, a serial and a console port and a monitor socket, no display. Changing it means changing NaN.
- A host needs Linux on x86_64, a C compiler as `cc` for the host tests, QEMU 10.2 or later, OVMF and swtpm.
- A change that makes a task use more stack than its bound fails the final build with `limit.stack-exceeded`, and raises the bound then.
- The boot heap bounds how many tasks the kernel runs at once; the instances that later tasks add draw from it too.
- Every defect or gap found in NaN goes to its repository rather than into a workaround; `docs/spec/22-nan.md` lists the features NaN OS requires, each with its issue.
