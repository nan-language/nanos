# 0082. Idle states

Status: accepted

## Context

Chapter 08 has a CPU with no work enter the deepest idle state whose exit latency is below half its predicted idle time and below its latency tolerance, else HLT. The prediction is the time to its earliest timer or, when shorter, the mean of its last 8 idle periods; the tolerance is the smallest `period_us` − `runtime_us` among the reservations admitted on it. The states are HLT and, on a machine with `mwait`, the MWAIT states of `PowerConfig.idle_states`. NaN's policy hook `idle` receives the CPU and its earliest timer and returns `Halt`, `Poll` or `Mwait(hint)`, with interrupts disabled and without allocating; the CPU calls it again each time it returns from an idle state without finding work. The runner, which targets `any`, admits reservations and cannot reach the kernel's scheduler state.

The sandbox's processors, QEMU's `max` model under TCG, have `mwait`, but QEMU ends each of their MWAITs at once, so a CPU idling there spins: an idle sandbox holds four host cores instead of 4% of one, and the store's emulated disk then faulted under host load (nan-language/nan#202).

## Decision

- **Choice.** The scheduler's state keeps, for each CPU, the lengths of its last 8 idle periods, each from its entry into an idle state to its return, and its latency tolerance. Its `idle` hook returns the deepest MWAIT state below both bounds, else `Halt`, and `/live/cpus` shows `hlt` or `mwait cN`, N the C-state the hint's upper nibble names.
- **Tolerance.** `runner::Machine`, what the runner takes from the machine, has `tolerate(cpu, tolerance_ns)`. After each admission and release, the runner calls it with that CPU's least slack, and the kernel writes it into the scheduler's state.
- **States.** The kernel gives the scheduler no MWAIT state while nan-language/nan#202 stands.

Alternatives rejected:

- **Admission in the scheduler's state:** each admission would cross into the kernel and the 80% rule would leave the runner, whose host tests exercise it, while the scheduler needs only each CPU's tolerance.
- **Giving the MWAIT states now:** every sandbox would spin its processors until NaN's emulator stops ending MWAIT at once.

## Consequences

- Every CPU idles in HLT.
- A tolerance applies to the CPU where its reservation was admitted, even though that CPU does not yet run the reservation's `latency` work.
