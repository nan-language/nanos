# Scheduling, time and power

This chapter defines run queues, CPU classes, shares and reservations, deadlines, wake placement, accounts and their accounting, clocks, timers, idle states and the idle machine, frequency, energy budgets, and power off and reboot.

## Run queues

Each CPU has one run queue (requires [N8](22-nan.md#n8)). It holds **entities**, one per [account](#accounting) with ready work on that CPU, and an entity runs its ready work in FIFO order.

- A **dispatch** is one handler call of an [instance](06-execution.md#instances) by the [runner](06-execution.md#runner). It is ready work of the account its message carries.
- A task is ready work of the account it was spawned under: a keyed task, the account of the dispatch that spawned it; a [job](06-execution.md#jobs)'s tasks, the job's account. Core tasks form an entity charged to no account, which runs after `latency` entities and before all others.
- A [space thread](04-kernel.md#user-entry) is ready work of the account of the domain its last `run` named.
- At each scheduling point of a CPU, when the running work ends or waits, a slice ends or a woken entity preempts, the [kernel policy](04-kernel.md#kernel-policies) `sched.pick` chooses the entity to run.

## CPU classes

| Class | Runs | Order | Slice |
| --- | --- | --- | --- |
| `latency` | before every other class, while its reservation has runtime left in the current period | earliest deadline first | the reservation's runtime |
| `normal` | when no `latency` entity is eligible | earliest virtual deadline among the eligible entities (EEVDF), by weight | 3 ms, at least 0.75 ms |
| `background` | only when the CPU has no `latency` or `normal` entity | the same as `normal` | 3 ms |

- An entity receives CPU time in proportion to its weight among the entities of its class on its CPU. A woken entity whose virtual deadline is earlier than the running one's preempts it once the running one has run 0.75 ms.
- A `latency` entity whose reservation has spent its runtime runs as `normal`, with its weight, and returns to `latency` when its next period begins.
- A CPU with a single entity arms no slice timer.

## Shares and reservations

A **share** is an account's entitlement to CPU time and energy:

```nan
pub closed enum CpuClass derives Copy, Eq, Encode {
    Latency
    Normal
    Background
} CpuClass

pub closed struct Share derives Copy, Eq, Encode {
    pub class: CpuClass
    pub weight: u16 where weight >= 1 and weight <= 10_000
    pub runtime_us: u32
    pub period_us: u32
    pub energy_mj_per_h: u64
    invariant runtime_us <= period_us
    invariant (class != CpuClass.Latency) or (period_us >= 1_000 and period_us <= 1_000_000 and runtime_us >= 50)
} Share
```

- By default the class is `Normal` (`Share.class`), the weight 100 (`Share.weight`), there is no reservation (`Share.runtime_us` and `Share.period_us` are 0) and no energy budget (`Share.energy_mj_per_h` is 0).
- An instance's share is the `share` field of its specification ([`/sys`](13-tree.md#sys)); a session's account takes the share of its principal's `Session` grant ([sessions as authority](09-authority.md#sessions-as-authority)), which the principal's sessions hold together: on each CPU, the entities of its sessions divide the grant's weight equally.
- Each account's executor carries its share. `Executor.split(Share)` returns a `CpuBudget` whose weight, reservation runtime and energy budget it moves from the splitting account's share, returns them when the `CpuBudget` closes, and fails when the splitting account lacks them (requires [N8](22-nan.md#n8)).
- A **reservation** is the `runtime_us` of CPU time in every `period_us` that a `latency` share gives.

A reservation is admitted when its instance starts or its `Session` grant is written, on one CPU: the one whose admitted reservations use the smallest fraction of it, the lowest numbered on ties, provided the reservations admitted there then use at most 80% of it. The [preview](14-changes.md#preview) of the plan reports the CPU it gets; a plan whose reservation fits on no CPU is refused ([`budget.over-reservation`](18-link.md#codes)). The `latency` work of the account, or of the principal's sessions together, runs only on that CPU, and the reservation ends when its instance stops or its grant ends.

## Deadlines

- A task's deadline is its earliest enclosing `task::within` deadline ([NaN 16 §Deadlines and races](https://github.com/nan-language/nan/blob/main/docs/spec/16-concurrency.md#deadlines-and-races)) (requires [N9](22-nan.md#n9)) or, for a keyed task without one, the deadline of the dispatch that spawned it. A dispatch's deadline is its message's.
- A message carries a deadline in its [header](07-communication.md#headers): the deadline of the dispatch or task that sent it, or none.
- **Inheritance**: an instance's next dispatch takes the earliest deadline among the messages in its intake and runs in the highest class among their accounts. In `latency` it consumes the reservation of the account whose message holds that deadline, while its CPU time is charged as [accounting](#accounting) states.
- In `latency`, deadlines order the entities; an entity without one uses the end of its reservation's current period. Deadlines never change the order of `normal` or `background` entities.

## Wake placement

The kernel policy `sched.place` chooses the CPU of every wake. By default:

1. A `Port.call` ([calls](07-communication.md#calls)), or a `Port.post` whose sending task then waits, runs the receiver's dispatch at once on the sender's CPU, and the reply switches back the same way (direct switch). When the receiver's dispatch is running on another CPU, or its reservation belongs to another CPU, the wake follows step 2.
2. Any other wake goes to the CPU the work last ran on when that CPU is idle, else to an idle CPU sharing that CPU's last-level cache, else to the waker's CPU.
3. A `latency` entity wakes on its reservation's CPU.

- A CPU becoming idle takes half the entities of the busiest run queue sharing its last-level cache, or, when none has any, of its NUMA node, or else of the machine; it never takes a `latency` entity.
- A wake onto another CPU sends a wake IPI only when that CPU is idle, runs a single entity or has to preempt (requires [N8](22-nan.md#n8)).
- On a machine whose measured fact `smt` holds ([measured facts](01-machines.md#measured-facts)), sibling hardware threads of one core run at the same time only work of the shared space or work of one principal's isolated space ([costs of isolation](09-authority.md#costs-of-isolation)).

A direct switch costs two task switches and no run-queue operation.

## Accounting

An **account** is the unit that CPU time, I/O and energy are charged to: one per instance and one per session. Its number is the 4-byte account field of the [message header](07-communication.md#headers), and `/live/accounts` publishes its counters ([`/live`](13-tree.md#live)).

- A dispatch is charged to the account its message carries, that of the session or instance that caused it; a task, to the account it runs under ([Run queues](#run-queues)).
- In an isolated space, a space thread's time is charged to the account of the instance its last `run` named ([system calls](04-kernel.md#system-calls)); a message from an isolated space carries its sender's account ([isolated transport](07-communication.md#isolated-transport)).
- Time in interrupt fns, landing fns and core tasks is charged to no account; `/live/cpus` reports it per CPU.
- The core reads the clock once per dispatch and per task switch, adds the elapsed time to a per-CPU accumulator, and folds the accumulator into the account at each switch.
- The driver or the [egress gate](16-network.md#egress-gate) that performs a transfer charges its bytes to the account of the message that requested it.
- Energy is CPU time × the CPU's power estimate. The estimate is calibrated from RAPL or the AArch64 activity monitors, at most once per 10 ms per running CPU, and without them is `PowerConfig.busy_mw`.
- Units: CPU time in ns, I/O in bytes, energy in mJ.
- Memory is never charged by message: it stays with the cell of the domain that owns it ([cells and limits](05-memory.md#cells-and-limits)).

## Clocks

### Monotonic clock

- Monotonic time counts nanoseconds from kernel entry ([time](02-identifiers.md#time)).
- On a machine with `invariant-tsc`, its source is the TSC, at the frequency CPUID leaf 0x15 reports or, when it reports none, as measured against the HPET or the ACPI PM timer over 10 ms at [start](04-kernel.md#start-sequence) step 6. A CPU whose TSC differs from the boot CPU's by more than 1 µs at start makes `invariant-tsc` absent.
- Without `invariant-tsc`, the source is the HPET, else the ACPI PM timer. On AArch64 it is `CNTVCT_EL0` at the frequency `CNTFRQ_EL0` gives.
- The kernel's `sys::Clock` reads this clock, and the kernel executor records it (requires [N9](22-nan.md#n9)).

### Wall clock

- `clock` keeps wall time as monotonic time plus an offset, with an uncertainty ([time](02-identifiers.md#time)), and gives it through `WallClock` capabilities ([capability kinds](09-authority.md#capability-kinds)).
- Its sources are the RTC, read at start with an uncertainty of 1 s; `set`, signed by the owner ([`clock`](20-base.md#clock)); and network time sources through `net` on a machine with `nic`.
- The uncertainty grows by 50 µs per second since the last reading and drops to the reading's own with each new one.
- A correction smaller than 128 ms slews the offset at 500 µs per second; a larger one steps it. When a source with an uncertainty below 1 s moves the time by more than 1 s, `clock` writes the RTC.
- Neither clock changes with the store: no revert, merge or branch switch moves them.

### Resolution

- Every `sys::Clock` and `WallClock` capability has a **resolution**, `ns`, `us` or `ms`, which an attenuation only coarsens ([capability kinds](09-authority.md#capability-kinds)) (requires [N9](22-nan.md#n9)). Its readings are truncated to a multiple of the resolution, and its sleeps and timers end at the first multiple at or after their deadline.
- Jobs of provenance `agent` get `ms` by default ([jobs](06-execution.md#jobs)).
- User mode reads no counter directly ([user entry](04-kernel.md#user-entry)), so resolution holds in isolated spaces.

## Timers

- Each CPU keeps one timer heap and programs its one-shot timer for the heap's earliest expiry: TSC-deadline on a machine with `tsc-deadline`, else the local APIC timer in one-shot mode, and `CNTV_CVAL_EL0` on AArch64 (requires [N8](22-nan.md#n8)). No periodic tick exists.
- The CPU reprograms its timer only when the earliest expiry moves earlier or fires.
- A timer's slack follows the class of the account that arms it: `latency` 0; `normal` 1% of its delay, at least 50 µs and at most 100 ms; `background` 10% of its delay, at most 1 s. A timer fires at its deadline or within its slack after it, together with every other timer whose window overlaps.
- A `normal` timer armed 10 s ahead fires within 100 ms after its deadline; one armed 1 ms ahead, within 50 µs.
- Timers serve `Cx.after` messages ([NaN 26 §Component context](https://github.com/nan-language/nan/blob/main/docs/spec/26-components.md#component-context)), task deadlines and sleeps, slice ends, reservation periods and the core's own periodic work. A timer of a hibernated instance stays armed and revives the instance when it fires.

## Idle

### Idle states

- A CPU with no entity enters the idle state that the kernel policy `power.idle` chooses: by default the deepest whose exit latency is below half the predicted idle time and below the CPU's latency tolerance, else HLT, or WFI on AArch64.
- The predicted idle time is the time to the CPU's earliest timer or, when shorter, the mean of its last 8 idle periods.
- The latency tolerance is the smallest `period_us` − `runtime_us` among the reservations admitted on the CPU, and unbounded when none is.
- The idle states are HLT, or WFI, and, on a machine with `mwait`, the MWAIT states of `PowerConfig.idle_states`.
- Before sleeping, a CPU takes work from other CPUs ([wake placement](#wake-placement)) and refills its zeroed frames ([frames](05-memory.md#frames)).

### Idle machine

- The machine is **idle** when no CPU has an entity and no timer is due within 1 s.
- On an idle machine, NaN OS's own work arms no timer and writes nothing to storage beyond the idle costs its mechanisms state ([mechanism costs](21-performance.md#mechanism-costs)). Checkpoint epochs run only while an instance is dirty ([durability](06-execution.md#durability)), metrics only for active accounts ([metrics](17-observability.md#metrics)), audit anchoring only after an unanchored entry ([audit chain](17-observability.md#audit-chain)), index flushes only after log growth ([index flushes](12-store-format.md#index-flushes)), energy calibration only on running CPUs, and the [watchdog](04-kernel.md#watchdog) is stopped.
- An expiry that no session or instance waits for, such as history thinning or the removal of a kept result, runs at the next wake-up with another cause.
- Every wake-up of an idle machine comes from a device interrupt or from a timer that an instance or a session armed. [21-performance](21-performance.md) bounds their count.

## Frequency

- On a CPU with hardware-controlled performance states (HWP), the kernel policy `power.perf` sets the energy-performance preference, by default the lowest preference among the classes the CPU runs.
- The core writes a CPU's preference only when it changes, and at most once per 10 ms.
- On other CPUs, the firmware's setting stays.

The record `PowerConfig` at [`/sys/config/power`](13-tree.md#sysconfig) configures these rules:

| Field | Meaning | Default |
| --- | --- | --- |
| `PowerConfig.idle_states` | MWAIT states, each with its hint and exit latency in µs | MWAIT C1, exit latency 1 µs |
| `PowerConfig.preference_latency` | preference for `latency` work, 0 to 255 | 0 |
| `PowerConfig.preference_normal` | preference for `normal` work | 128 |
| `PowerConfig.preference_background` | preference for `background` work | 255 |
| `PowerConfig.busy_mw` | power of a running CPU without a calibration source, in mW | 2,000 |

## Energy

An account's **energy budget** is `Share.energy_mj_per_h` millijoules per hour: a bucket that holds at most one hour of budget and refills continuously at the budget ÷ 3,600 per second. The accounts of a principal's sessions drain one bucket, and its state applies to each of them.

| State | Event | Next | Effect |
| --- | --- | --- | --- |
| within budget | the bucket empties | throttled | the account's entities run as `background` |
| throttled | the bucket refills to one minute of budget | within budget | the account's entities run in their class again |

`/live/accounts` shows each account's energy and state. An account without an energy budget is never throttled.

## Power off and reboot

`off` and `reboot` are operations of [`power`](20-base.md#power) of class `restorable`. Both run these steps:

1. The runner pauses the intake of every instance except the base instances the next steps use, checkpoints every dirty checkpointed instance ([durability](06-execution.md#durability)), and waits for the store's index flush ([index flushes](12-store-format.md#index-flushes)). A checkpoint that fails leaves its instance's previous checkpoint the latest ([checkpoints](06-execution.md#checkpoints)), and an index flush that fails leaves the log to replay at the next mount ([mounting and recovery](12-store-format.md#mounting-and-recovery)); neither stops the operation.
2. The core stops the [watchdog](04-kernel.md#watchdog).
3. For `off`, the core writes the sleep type of `\_S5`, which `devices.aml` evaluates ([device discovery](15-devices.md#device-discovery)), to the FADT's PM1 control registers, or to its sleep control register on a hardware-reduced ACPI machine; on AArch64 it calls PSCI `SYSTEM_OFF`. For `reboot`, it writes the FADT's reset register, or calls PSCI `SYSTEM_RESET`. In the sandbox, `off` writes byte 0 to the exit port and `reboot` byte 2 ([the sandbox](01-machines.md#the-sandbox)).
4. Failure: when the machine still runs 5 s after the write, the core arms the watchdog again, the intakes reopen, and the operation fails with [`fault.power-failed`](04-kernel.md#codes).
