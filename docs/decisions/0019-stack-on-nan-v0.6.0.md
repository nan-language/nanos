# 0019. Stack on NaN v0.6.0

Status: accepted

## Context

NaN OS is an operating system built for AI agents from the root. Agents write it, so its language must be one they write and read reliably, and every change must be observable without real hardware. It must also work on the machines agents run on, many of which are AArch64.

Release v0.6.0 gives a kernel what chapter 15 drives devices with: PCI functions whose windows, interrupts, power and reset derive from them without a trusted premise and which a keeper takes back cut off when their holder releases them or its domain is settled, PCI ports with native hot plug, I/O domains, linear Envs, and runner capabilities through which a component postpones a request and answers it from a later handler. Its sandbox machine adds an NVMe namespace, a SATA disk on the AHCI controller, an empty hot-plug PCIe root port and a QMP monitor socket. `sys::Wire.serve` numbers each request and gives no reply for a postponed one.

## Decision

- NaN, language version 1, with its toolchain `nan` at release v0.6.0, pinned by `tools/install`.
- The toolchain is the only build tool: `nan check`, `nan fmt`, `nan test`, `nan build` and `nan run`.
- The repository root is one NaN workspace, each component a package: `loader`, the boot loader, for `x86_64-uefi`, `kernel` for `x86_64-bare`, and the libraries they use for `any`, whose tests run on the host.
- QEMU with OVMF and swtpm, as NaN's target descriptors fix them, runs every image and every test of the machine targets, with KVM where the host offers it.
- The specification in `docs/spec/` defines NaN OS for x86_64 and AArch64. Architecture-specific code lives in NaN target variants (`name@target`), so the rest is shared. The x86_64 machine is the one the sandbox runs.

Alternatives rejected:

- **Staying on v0.4.0:** a driver would map its device's registers under a trusted premise, nothing would cut a device off when its driver traps, and a driver could answer a request only by returning, before its device completes it.

## Consequences

- Agents learn NaN from `nan primer` rather than from training.
- The machine is the one the descriptors fix: q35 with an IOMMU, 4 processors, 512 MiB, an entropy device, a TPM, a watchdog, a virtio disk, an NVMe namespace, a SATA disk, an empty hot-plug slot, a NIC, a serial and a console port and a monitor socket, no display. Changing it means changing NaN.
- A host needs Linux on x86_64, a C compiler as `cc` for the host tests, QEMU 10.2 or later, OVMF and swtpm.
- Every defect or gap found in NaN goes to its repository rather than into a workaround; `docs/spec/22-nan.md` lists the features NaN OS requires, each with its issue.
