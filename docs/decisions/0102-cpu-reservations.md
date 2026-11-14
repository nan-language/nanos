# 0102. CPU reservations

Status: accepted

## Context

Chapter 08 admits a `latency` share's reservation, `runtime_us` in every `period_us`, when its instance starts or its `Session` grant is written, on the CPU whose admitted reservations use the smallest fraction of it, the lowest numbered on ties, provided they then use at most 80% of it; a reservation fitting on no CPU is refused with `budget.over-reservation`, and it ends when its instance stops or its grant ends. The account's `latency` work runs only on that CPU, before every other class while the reservation has runtime left in the current period, earliest deadline first, work without a deadline taking its period's end; once the runtime is spent it runs as `normal` until the next period. A CPU's latency tolerance is the least slack among the reservations admitted on it. The scheduler's state keeps each account's terms (ADR 0101).

## Decision

- **Admission.** The runner keeps the millionths of each CPU its admitted reservations use and numbers each reservation it admits. A start of an instance whose share is `latency` admits its reservation unless it holds one, and fails with `budget.over-reservation` when the least used CPU cannot take it within 80%.
- **End.** An instance's reservation ends with the incarnation's domain, except at a hibernation, and when the start fails after admitting it; the next start admits it again.
- **Running.** `runner::Machine` has `run_reservation` and `end_reservation`, by number. Each CPU keeps the reservations it runs, renewing a reservation's runtime as a later period begins and charging it the `latency` work that draws on it.
- **Placement.** `latency` work drawing on a reservation goes to its CPU, bound there, keyed by its deadline or its period's end ahead of core work while the period has runtime left, its slice the runtime left when other work waits; past that, it is keyed by its account's virtual deadline as `normal` work. `latency` work drawing on none runs as `normal`.
- **Tolerance.** A CPU's latency tolerance is the least slack among the reservations it runs.

Alternatives rejected:

- **Admission in the scheduler's state:** each admission would cross into the kernel and the 80% rule would leave the runner, whose host tests exercise it.

## Consequences

- A reservation renews its runtime only when its CPU queues or charges its work, which needs no timer.
- A plan's preview reports the CPU each reservation gets (ADR 0083).
