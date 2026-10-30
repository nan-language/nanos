# Machines

This chapter defines the architectures NaN OS runs on, the minimum machine, the machine conditions with how the kernel detects each and what replaces it when absent, the measured facts and the defaults derived from them, and the sandbox machine with the rules it exercises.

## Architectures

A **machine** is the hardware NaN OS runs on: an x86_64 or an AArch64 machine. Each program of NaN OS is built for one NaN target per architecture:

| Program | x86_64 | AArch64 |
| --- | --- | --- |
| [loader](03-boot.md#loader) | `x86_64-uefi` | `aarch64-uefi` |
| [kernel image](04-kernel.md#kernel-image) | `x86_64-bare` | `aarch64-bare` |
| [bundles](10-code.md#artefacts-and-bundles) and [jobs](06-execution.md#jobs) | `x86_64-nanos` | `aarch64-nanos` |

- Architecture-specific modules are target variants `name@target` ([NaN 03 §Target variants](https://github.com/nan-language/nan/blob/main/docs/spec/03-packages-modules.md#target-variants)).
- Bundles and jobs are built for the nanos targets (requires [N1](22-nan.md#n1)).
- On AArch64, NaN OS takes the memory layout, the UART, the interrupt controller and the PSCI conduit from the device tree or ACPI, not from the values NaN's AArch64 descriptors fix (requires [N30](22-nan.md#n30)).

## Minimum machine

| Requirement | x86_64 | AArch64 |
| --- | --- | --- |
| CPU | x86-64 with SSE2 and the NX bit | Armv8.0-A with floating point and Advanced SIMD |
| Translation | 4-level paging | 4 KiB granule, 48-bit virtual addresses |
| Interrupt controller | local APIC and I/O APIC | GICv3 |
| Counter | the TSC, and an HPET or an ACPI PM timer when the TSC is not invariant | the generic timer |
| CPU control | — | PSCI |
| Firmware | UEFI with boot services | UEFI, or a program that enters the kernel image with a device tree ([direct boot](03-boot.md#direct-boot)) |
| Description | ACPI | ACPI or a device tree |
| Memory | 64 MiB of RAM, usable at the kernel image's physical addresses | the same |
| Storage | a boot volume on a block device of a class the base drivers serve ([15-devices](15-devices.md#drivers)) | the same |
| Link | one [transport](18-link.md#transports): a serial port, virtio-vsock or a NIC | the same |

Every other hardware feature a rule depends on is a machine condition.

## Machine conditions

A **machine condition** is a hardware feature some rules depend on. A machine lacking one has the outcome its row gives.

| Condition | Holds when | Without it |
| --- | --- | --- |
| `iommu` | an ACPI DMAR or IVRS table, or an IORT or device-tree node naming an SMMUv3, describes IOMMUs, and the kernel enables them before any driver starts ([04-kernel](04-kernel.md#start-sequence)) | DMA is unconfined and the device trust rule applies ([09-authority](09-authority.md#machine-conditions)) |
| `tpm` | the loader finds `EFI_TCG2_PROTOCOL` reporting a TPM 2.0 whose SHA-256 PCR bank is active | no measurement and no volume encryption ([12-store-format](12-store-format.md#volumes)); `keys` holds its keys in software ([09-authority](09-authority.md#keys-on-the-machine)); the floor and the owner key live in UEFI variables only ([03-boot](03-boot.md#floor-and-owner-key-storage)) |
| `secure-boot` | the loader reads the UEFI variables `SecureBoot` as 1 and `SetupMode` as 0 | the firmware starts any loader, so a replaced loader goes undetected |
| `user-mode` | the kernel image holds the entry of isolated spaces (requires [N23](22-nan.md#n23)), and the [user window](04-kernel.md#isolated-spaces) starts at or below 2^46 | no isolated space exists, and placement refuses every instance it places isolated, except `keys` ([09-authority](09-authority.md#machine-conditions)) |
| `invariant-tsc` | CPUID leaf `0x8000_0007` EDX bit 8; always on AArch64 | monotonic time comes from the HPET, else the ACPI PM timer ([08-scheduling](08-scheduling.md#clocks)) |
| `tsc-deadline` | CPUID leaf 1 ECX bit 24; always on AArch64 | one-shot timers use the local APIC timer in one-shot mode ([08-scheduling](08-scheduling.md#timers)) |
| `pcid` | CPUID leaf 1 ECX bit 17; always on AArch64, which has ASIDs | each switch between spaces flushes the CPU's non-global translations ([04-kernel](04-kernel.md#isolated-spaces)) |
| `pages-1g` | CPUID leaf `0x8000_0001` EDX bit 26; always on AArch64 | the identity map uses 2 MiB pages ([04-kernel](04-kernel.md#address-spaces)) |
| `mwait` | CPUID leaf 1 ECX bit 3; never on AArch64 | idle CPUs wait with HLT, on AArch64 with WFI ([08-scheduling](08-scheduling.md#idle)) |
| `watchdog` | an ACPI WDAT table, an Intel TCO watchdog whose NO_REBOOT bit the kernel can clear, or an SP805 the device tree or ACPI lists | a halted machine stays stopped ([03-boot](03-boot.md#reboot-after-a-halt)) |
| `nic` | a NIC of medium `ethernet` ([Nic](15-devices.md#nic)) is bound to a driver of the base set ([15-devices](15-devices.md#drivers)) | the link runs on serial and vsock transports only ([18-link](18-link.md#transports)) |
| `meltdown-immune` | `IA32_ARCH_CAPABILITIES` bit 0 (RDCL_NO) is 1, or the CPU vendor is AMD or Hygon; on AArch64, `ID_AA64PFR0_EL1.CSV3` is 1 | an isolated space's user code runs on a root mapping only its user window and the entry trampoline ([split tables](04-kernel.md#split-tables)) |

- The loader detects `tpm` and `secure-boot` (requires [N14](22-nan.md#n14)) and passes them in the [handoff record](03-boot.md#handoff-record). The kernel detects the others except `nic` at start, before it starts any instance, reading CPUID with the `cpuid` intrinsic (requires [N11](22-nan.md#n11)) and `IA32_ARCH_CAPABILITIES` (MSR `0x10A`) only when CPUID leaf 7 EDX bit 29 enumerates it.
- `nic` holds while `devices` has such a NIC bound. Every other condition is fixed for the boot.

## Measured facts

The kernel measures these facts at start and publishes them, with the conditions, as the `Machine` record at `/live/machine` ([13-tree](13-tree.md#live)).

| Fact | Field | Measured from |
| --- | --- | --- |
| architecture | `arch` | the kernel image's target |
| conditions | `conditions` | [machine conditions](#machine-conditions), one field per condition, its hyphens written `_` |
| RAM | `ram_bytes` | the usable memory of the memory map the kernel receives |
| CPUs | `cpus` | the CPUs the platform reports usable ([NaN 24 §Anchor](https://github.com/nan-language/nan/blob/main/docs/spec/24-kernel.md#anchor)) |
| level | `level` | the highest [level](10-code.md#bundle-file) whose features the CPU reports |
| SMT | `smt` | whether a core runs two or more hardware threads: CPUID leaf `0xB` on x86_64; on AArch64, `MPIDR_EL1.MT` and the CPU topology the device tree or ACPI gives |
| NUMA nodes | `numa_nodes` | the proximity domains of the ACPI SRAT, or the `numa-node-id` values of the device tree; 1 without either |
| storage | `storage` | the device holding the [store](12-store-format.md#placement): `Nvme` for an NVMe namespace, `Ssd` for an ATA device reporting a nominal media rotation rate of 1, `Rotational` for another ATA device, `Flash` for SD and eMMC, `Virtual` for virtio-blk, `File` for the store file on the boot volume |
| power | `power` | `Battery` when ACPI lists a battery device (`_HID` `PNP0C0A`) or the device tree a `simple-battery` node; else `Mains` |

```nan
pub closed enum Arch derives Copy, Eq, Render, Encode {
    Amd64
    Aarch64
} Arch

pub closed struct Conditions derives Copy, Eq, Render, Encode {
    pub iommu: bool
    pub tpm: bool
    pub secure_boot: bool
    pub user_mode: bool
    pub invariant_tsc: bool
    pub tsc_deadline: bool
    pub pcid: bool
    pub pages_1g: bool
    pub mwait: bool
    pub watchdog: bool
    pub nic: bool
    pub meltdown_immune: bool
} Conditions

pub closed enum Storage derives Copy, Eq, Render, Encode {
    Nvme
    Ssd
    Rotational
    Flash
    Virtual
    File
} Storage

pub closed enum PowerSource derives Copy, Eq, Render, Encode {
    Mains
    Battery
} PowerSource

pub closed struct Machine derives Copy, Eq, Render, Encode {
    pub arch: Arch
    pub conditions: Conditions
    pub ram_bytes: u64
    pub cpus: u32
    pub level: Level
    pub smt: bool
    pub numa_nodes: u32
    pub storage: Storage
    pub power: PowerSource
} Machine
```

`Level` is the type of [bundle file](10-code.md#bundle-file). Every fact is fixed for the boot, except `conditions.nic`.

## Derived defaults

A default depends only on measured facts and machine conditions, never on the name of a machine, a board or a profile. Each rule states its default where it applies:

| Default | Facts | Rule |
| --- | --- | --- |
| on-device admission with the `nan` job | RAM, the jobs the generation ships | [20-base](20-base.md#jobs) |
| checkpoint epoch | power, storage | [06-execution](06-execution.md#durability) |
| memory watermarks and the core reserve | RAM, CPUs | [05-memory](05-memory.md#pressure) |
| frame pools | NUMA nodes | [05-memory](05-memory.md#frames) |
| wake placement and stealing | CPUs, NUMA nodes | [08-scheduling](08-scheduling.md#wake-placement) |
| core scheduling between principals | SMT | [08-scheduling](08-scheduling.md#wake-placement) |

## The sandbox

The sandbox is the machine `tools/sandbox start` boots ([sandbox](../sandbox.md)): the emulator machine of NaN's `x86_64-uefi` descriptor ([NaN 27 §Descriptors](https://github.com/nan-language/nan/blob/main/docs/spec/27-program-model.md#descriptors)).

- A q35 machine with 4 CPUs and 512 MiB of RAM, on QEMU's default x86_64 CPU model, `qemu64`, accelerated by KVM when the host offers it.
- OVMF without secure boot, whose UEFI variables last for the run.
- An AHCI disk holding the [boot volume](03-boot.md#boot-volume) as a virtual FAT volume made from a host directory at each start: the development loader at `/EFI/BOOT/BOOTX64.EFI`, the kernel image at `/kernel.elf`, and the boot record and the store file the machine writes.
- A virtio-rng device.
- The 16550 serial port, on the sandbox's standard output and standard input (requires [N22](22-nan.md#n22)), carrying the console and the link's serial transport.
- The exit port: writing the byte v ends the run with status v mod 128. [Power off and reboot](08-scheduling.md#power-off-and-reboot) write it, and a halt writes 70; a reset ends the run with status 70, so the sandbox never reboots.

Its facts are `arch: Arch.Amd64`, `cpus: 4`, `level: Level.X86v1`, `smt: false`, `numa_nodes: 1`, `storage: Storage.File` and `power: PowerSource.Mains`. No condition holds except `meltdown-immune`, which holds unless KVM runs the sandbox on a CPU whose vendor is neither AMD nor Hygon: `qemu64` reports AMD as its vendor, and KVM substitutes the host's. Without `user-mode`, `meltdown-immune` changes no rule there.

The sandbox exercises:

- the outcome of each absent condition: 2 MiB pages, HPET time, the local APIC one-shot timer, HLT, unconfined DMA, no measurement, no isolated space, no watchdog and a link on serial only;
- the development loader and `dev` mode, the loader's path without a valid boot record, the handoff record, and entropy from virtio-rng;
- halts, power off and reboot, through the exit port.

It exercises no rule that needs a condition, a reboot or state kept across runs: trial boots and the health gate, generation switches, recovery, the floor, owner key requests, loader replacement, measurement, DMA confinement, isolated spaces with their split tables and PCIDs, 1 GiB pages, TSC time and deadlines, MWAIT, the watchdog, network transports and AArch64. These rules need an emulator with those devices and CPU features, reboot and persistence (requires [N22](22-nan.md#n22)).
