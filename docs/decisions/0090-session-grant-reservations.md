# 0090. Session grant reservations

Status: accepted

## Context

Chapter 08 admits the reservation of a `latency` share when its instance starts or its `Session` grant is written, ends it when the instance stops or the grant ends, and has the plan's preview report the CPU it gets or refuse the plan. The runner admits instances' reservations; the grants written to `/sys/grants` reach `link`, which reads them for the sessions it opens, and not the runner. A preview's effect set reports the CPU of an instance's reservation on its activation and has no field for a grant's.

## Decision

- **Hand-over.** `link` reads `/sys/grants` at its start, at each change of it and at the earliest expiry among the `Session` grants asking for a reservation, and gives the core, through `Accounts.reserve`, every such grant that is live. The runner admits the grants it holds no reservation for, ends those of grants left out or asking for another, and answers the reservation each holds.
- **Previews.** A preview admits the reservation of each `Session` grant the plan writes after the instances its activations start, as the core holds the instances' and the grants' reservations, the grant's own released first; `Widening.cpu` reports the CPU, and a reservation that fits on no CPU refuses the plan.

## Consequences

- A grant's reservation follows the grant only while `link` runs; a restarted `link` hands every grant over again.
- `link` keeps the reservations the core answered, which its previews admit against.
