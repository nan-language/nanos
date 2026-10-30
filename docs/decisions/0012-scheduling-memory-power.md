# 0012. Scheduling, memory and power without ticks, swap or overcommit

Status: accepted

## Context

NaN OS must be as efficient as today's operating systems, from a 64 MiB board to a many-core server. NaN's kernel executor has one global run queue and a periodic tick (NaN ADR 0035). An AI controlling the machine is its slowest component, with decisions taking seconds, so it must stay off every hot path. Agent workloads are requests crossing many components, whose cost belongs to whoever caused them.

## Decision

- **Scheduling.** Per-CPU run queues with stealing.
  - Classes: `normal` (EEVDF by weight), `background`, and `latency` (EDF within admitted reservations).
  - Deadlines from `task::within` travel with messages, so a service inherits the urgency of the request it serves.
  - Synchronous calls hand off on the caller's CPU.
- **Accounting.** CPU time, I/O and energy are charged to the account of the request being handled. Memory stays with the instance that owns it.
- **Timers.** No periodic tick: one-shot timers with slack by class. An idle machine causes no wake-up and no storage write.
- **Memory.** Memory quotas are NaN cells, with limits and optional reserved regions.
  - There is no overcommit beyond limits, no demand paging and no swap.
  - Under pressure, which follows a watermark scaled by √RAM, the system drops clean caches, asks instances to shrink, then hibernates idle instances by encoding their state.
  - A draw fails where it is made, and there is no out-of-memory killer.
- **Agents.** They install reflexes (components and kernel policies) for routine events and sleep until an event matches their wake rules.

Alternatives rejected:

- **Swap and demand paging:** thrashing, and invisible latency.
- **The periodic tick and one run queue:** constant wake-ups and contention.
- **Fair share alone:** it cannot honour the deadlines requests carry.
- **RAPL reads on every switch:** an MSR read per handler, absent under KVM.
- **Profiles such as small, standard and server:** each default follows from measured facts instead.

## Consequences

- A hibernated instance costs about 2 KiB and revives in about 1 ms per 100 KiB of state.
- The policy needs an executor seam, per-CPU vectors, one-shot timers, intrinsics, and extended state saved only for float users ([N8](../spec/22-nan.md#n8), [N11](../spec/22-nan.md#n11), [N21](../spec/22-nan.md#n21), [N24](../spec/22-nan.md#n24)).
- The sandbox's CPU model lacks invariant TSC, TSC-deadline, PCID and MWAIT, so the pipeline exercises the fallback paths and measures primary paths on reference hardware ([21](../spec/21-performance.md)).
