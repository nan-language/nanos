# Performance

This chapter defines the machines that performance is measured on, the `bench` job and its report, the gating rules, every measurable requirement with its bound, machine and method, and the costs each mechanism states.

## Machines

| Machine | What it is |
| --- | --- |
| `build` | the image files `nan build` writes, with the final-build rules |
| `sandbox` | [the sandbox](01-machines.md#the-sandbox), accelerated by KVM, on an x86_64 host whose CPU runs at 3 GHz or more |
| `reference` | the **reference machine**: an x86_64 machine with every condition of [machine conditions](01-machines.md#machine-conditions), at least 4 cores at a base frequency of 3 GHz or more, 16 GiB of RAM, and an NVMe SSD on PCIe 3.0 ×4 or faster holding its boot volume and store |

A requirement whose machine is "both" holds on the sandbox and on the reference machine. A report names the reference machine's CPU model, microcode, firmware, RAM and SSD.

## Measurement

`bench` is a job of the generation ([jobs](20-base.md#jobs)) that measures the machine it runs on. Its input is a `BenchRequest`; its result a `BenchReport`:

```nan
pub closed struct BenchRequest derives Encode {
    pub rows: mem::List[text::String]
    pub runs: u32 where runs >= 1
    pub idle_s: u32
} BenchRequest

pub closed enum BenchMachine derives Encode {
    Sandbox
    Reference
} BenchMachine

pub closed enum RowKind derives Encode {
    Count
    Time
    Rate
} RowKind

pub closed struct BenchRow derives Encode {
    pub id: text::String
    pub kind: RowKind
    pub unit: text::String
    pub bound: u64
    pub runs: u32
    pub median: u64
    pub low: u64
    pub high: u64
    pub passed: bool
} BenchRow

pub closed struct BenchReport derives Encode {
    pub machine: BenchMachine
    pub hardware: text::String
    pub toolchain: HashValue
    pub kernel: HashValue
    pub generation: u64
    pub rows: mem::List[BenchRow]
} BenchReport
```

- `rows` names the requirements to measure, every one of the machine when empty; `runs` is the number of reported runs per row, and `idle_s` the duration of P5.
- `toolchain` is the hash of the NaN toolchain that built the kernel image, `kernel` the kernel image's blob id.
- `bench` runs on a machine running only the base instances and its own instances, its rows one after another. It starts 60 s after the health gate passes, or in `dev` mode 60 s after the boot's first ready record ([health gate](03-boot.md#health-gate)).
- A **time** row reads the monotonic clock in nanoseconds ([monotonic clock](08-scheduling.md#monotonic-clock)) around a batch of k operations, k making the batch last at least 10 ms, and divides by k. A **count** row reads counters of the core, the runner and the drivers before and after its workload. A **rate** row divides the bytes processed by a timed batch.
- Each row runs once unreported, then `runs` times. `median`, `low` and `high` are over those runs; values are integers in the row's unit.
- P1 is measured on the `kernel.elf` file by the build that writes it; `bench` reports it from the running kernel image.

## Gating

- A count row passes when every run is within its bound; a time row when its median is; a rate row when its median is at least its bound.
- The **pipeline** is the check every change to NaN OS passes before it merges. It boots the sandbox, runs `bench` with `runs` 11 and `idle_s` 300, and fails the change when a row of machine `sandbox` or both fails.
- A vendor must not sign a generation's manifest ([generations and manifests](03-boot.md#generations-and-manifests)) unless a report of that generation on the reference machine, with `runs` 11 and `idle_s` 3,600, passes every row of machine `reference` or both.

## Requirements

MB is 10^6 bytes.

| Id | Measure | Bound | Machine | Kind | Method |
| --- | --- | --- | --- | --- | --- |
| P1 | kernel image | ≤ 8 MiB | build | count | the bytes of `kernel.elf` ([kernel image](04-kernel.md#kernel-image)) |
| P2 | loader start to kernel entry, 8 MiB image | ≤ 50 ms | reference | time | (`kernel_entry` − `loader_entry`) ÷ `counter_hz` of `LiveBoot` ([/live](13-tree.md#live)): the time-stamp counter from the loader's entry ([handoff record](03-boot.md#handoff-record)) to the kernel's |
| P3 | loader start to link ready | ≤ 300 ms sandbox, ≤ 150 ms reference | both | time | P2's value plus `ready_ns` of `link`'s `LinkSummary`, the monotonic time of the boot's first ready record ([framing](18-link.md#framing)), booting a store holding the base instances only |
| P4 | store recovery | ≤ 16 MiB of log replayed | both | count | the largest log length after the latest durable flush record ([index flushes](12-store-format.md#index-flushes)), sampled every 10 ms during 60 s of 1 KiB commits sent as fast as the store takes them; on the reference machine also the `replayed` of `store.mounted` after a power cut during that load |
| P5 | idle machine | 0 wake-ups and 0 storage writes per hour (requires [N8](22-nan.md#n8)) | both | count | over `idle_s`, in the [idle configuration](#mechanism-costs): the timer expiries and IPIs of every CPU, and the write, flush and discard commands of every `Block` device ([Block](15-devices.md#block)) |
| P6 | task switch, integer state | ≤ 200 ns sandbox, ≤ 100 ns reference (requires [N24](22-nan.md#n24)) | both | time | half the round trip of two keyed tasks of one instance on one CPU, each waking the other through an `hw::Signal` |
| P7 | task switch, extended state | ≤ 300 ns | both | time | as P6, both tasks using floats |
| P8 | request and reply in the shared space, one CPU | ≤ 0.5 µs | both | time | `Port.call` of a `u64` request answered with a `u64`, between two instances of one bundle on one CPU ([calls](07-communication.md#calls)) |
| P9 | request and reply across CPUs, receiver's CPU running | ≤ 1.5 µs (requires [N8](22-nan.md#n8)) | reference | time | as P8, the server on another CPU that runs a `background` task |
| P10 | instance start from a loaded bundle | ≤ 50 µs | both | time | from `supervisor`'s start of an instance with an empty Env and 64 bytes of state to its `instance.started` ([starting](06-execution.md#starting)) |
| P11 | job start from an admitted bundle | ≤ 1 ms per MiB of code | both | time | from the activation of a `run` step to the first instruction of `main`, for bundles of 1 MiB and 4 MiB of code that no instance has loaded ([loading](10-code.md#loading)) |
| P12 | live update unavailability, 100 KiB of state | ≤ 2 ms, without the acknowledgement of cancelled I/O | both | time | from the pause of the intake to its reopening, less the wait for acknowledgements, for an update of class `identical` ([live update](10-code.md#live-update)) |
| P13 | revival from the RAM pool, 100 KiB of state | ≤ 1 ms | both | time | from the arrival of a message at a hibernated instance whose image the pool holds to its handler call ([hibernation memory](05-memory.md#hibernation-memory)) |
| P14 | durable commit | ≤ 2 device flushes and ≤ 50 µs of CPU | both | count, time | a transaction writing one 1 KiB record with no other writer: the flushes the `Block` driver receives per commit, and the CPU time charged to `store`'s account per commit ([commit procedure](11-store.md#commit-procedure)) |
| P15 | journal record from the core | ≤ 50 ns | both | time | a 32-byte record written into a core ring ([core rings](17-observability.md#core-rings)) |
| P16 | capability check | ≤ 15 ns; ≤ 5 ns (requires [N10](22-nan.md#n10)) | both | time | an operation through a Copy capability failing at its stale generation ([capability kinds](09-authority.md#capability-kinds)) |
| P17 | allocation | ≤ 45 ns | both | time | a box drawn through `mem::Alloc` with its cell charge, every box kept |
| P18 | idle base memory | ≤ 40 MiB on a machine with 4 CPUs, NaN's 16 MiB boot heap, the kernel image and the core reserve included | both | count | `ram` less `free`, `store_cache` and `blob_cache` in `/live/memory` ([memory summary](05-memory.md#memory-summary)), in the idle configuration |
| P19 | active instance footprint | one 64 KiB chunk | both | count | the chunks charged to the instance of P10 once started |
| P20 | hibernated instance footprint | ≤ 2 KiB | both | count | the bytes that instance holds resident once hibernated, its image excluded |
| P21 | interrupt to driver task | ≤ 6 µs sandbox, ≤ 1 µs reference | both | time | from the entry of the interrupt fn to the driver's keyed task running, for a `virtio-rng` completion in the sandbox and an NVMe completion on the reference machine ([interrupts and polling](15-devices.md#interrupts-and-polling)) |
| P22 | BLAKE3 per core | ≥ 250 MB/s; ≥ 1 GB/s (requires [N17](22-nan.md#n17)) | both | rate | hashing 64 MiB in memory on one CPU |
| P23 | `mem::wire` encode and decode | ≥ 200 MB/s each per core | both | rate | encoding, then decoding, a 1 MiB record of integers, texts and lists on one CPU |
| P24 | cleaner write amplification | ≤ 2 | reference | count | bytes the cleaner writes per byte of new log during 1 h of random 4 KiB overwrites on a store filled to 70% ([the cleaner](12-store-format.md#the-cleaner)) |

## Mechanism costs

Each chapter states, for each mechanism it defines:

- its **idle cost**: the timers it arms, the wake-ups it causes and the storage writes it makes on an [idle machine](08-scheduling.md#idle-machine), none unless it names the timer, its period and the condition that arms it;
- its **operation cost**: per message, commit or request, a time bound, or the count of copies, allocations, device flushes and wake-ups it adds.

A mechanism costs no more than its chapter states, and the rows above bound the mechanisms they name.

The **idle configuration** of P5 and P18 is a machine that runs only the base instances, holds no detached session or pending approval, passed the health gate, or in `dev` mode wrote its first ready record, 60 s before, and on the reference machine has static network addresses ([address configuration](16-network.md#address-configuration)) and no network time source ([`clock`](20-base.md#clock)). Device interrupts that deliver input are not counted.
