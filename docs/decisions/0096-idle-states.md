# 0096. Idle states

Status: superseded by 0103

## Context

Chapter 08 has a CPU with no work enter the deepest idle state whose exit latency is below half its predicted idle time and below its latency tolerance, else HLT. The prediction is the time to its earliest timer or, when shorter, the mean of its last 8 idle periods; the tolerance is the smallest `period_us` − `runtime_us` among the reservations admitted on it. The states are HLT and, on a machine with `mwait`, the MWAIT states of `PowerConfig.idle_states`, which `power` has the core apply with the rest of the power settings. NaN's policy hook `idle` receives the CPU and its earliest timer and returns `Halt`, `Poll` or `Mwait(hint)`, with interrupts disabled and without allocating. The runner, which targets `any`, admits reservations and cannot reach the kernel's scheduler state. Under TCG, NaN's machine runs QEMU's `max` model without `monitor`, so the sandbox has no `mwait`.

## Decision

- **Choice.** The scheduler's state keeps, for each CPU, the lengths of its last 8 idle periods, each from its entry into an idle state to its return, and its latency tolerance. Its `idle` hook returns the deepest MWAIT state below both bounds, else `Halt`, and `/live/cpus` shows `hlt` or `mwait cN`, N the C-state the hint's upper nibble names.
- **States.** On a machine with `mwait`, each CPU keeps under its lock the MWAIT states of the power settings applied last, at most 8, their exit latencies in ns; until the settings are applied, and on a machine without `mwait`, it keeps none.
- **Tolerance.** `runner::Machine`, what the runner takes from the machine, has `tolerate(cpu, tolerance_ns)`. After each admission and release, the runner calls it with that CPU's least slack, and the kernel writes it into the scheduler's state.

Alternatives rejected:

- **Admission in the scheduler's state:** each admission would cross into the kernel and the 80% rule would leave the runner, whose host tests exercise it, while the scheduler needs only each CPU's tolerance.
- **Enumerating the states from CPUID leaf 5:** it gives no exit latency, which the choice needs.

## Consequences

- Power settings holding more than 8 idle states fail `configure` with `schema.invalid`.
- Every CPU of the sandbox idles in HLT.
