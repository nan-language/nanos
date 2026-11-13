# 0083. Reservations in plan previews

Status: accepted

## Context

Chapter 08 has a plan's preview report the CPU a reservation gets, and refuse with `budget.over-reservation` a plan whose reservation fits on no CPU; chapter 05 refuses a plan whose reserved instances' limits sum to more than RAM less the core reserve. The runner admits reservations, as instances start, on the least used CPU within 80% of it; `link` previews plans and reaches the runner only through `Control` orders on single instances and through the tree.

## Decision

- **Rule.** The admission rule, which CPU a reservation goes to and within what share, is `os::admitting_cpu`, which the runner and `link` both apply.
- **State.** Each instance's record in `/live/instances` shows the reservation admitted for it, its CPU and the millionths of it it takes. A preview simulates admissions over those records: each activation that starts or restarts an instance releases the reservation it holds and admits the one its specification on the preview branch asks for, in the effect set's order, and its `activation` record shows the CPU. `link` reads `/live` and the running branch through its own View, and the specifications the plan writes through the previewing principal's View of the preview branch, which only the previewer may open.
- **Memory.** A plan writing the specification of a reserved instance has the limits of every reserved instance summed against RAM less the core reserve, as `/live/memory` and `/live/machine` tell: those of the specifications it writes as it writes them, the others as the running branch holds them.
- Plans for branches other than the running branch reserve nothing and are not checked.

Alternatives rejected:

- **A `Control` order asking the runner for every reservation:** an order names one instance, and the reservations are observed state that agents read in `/live` too.

## Consequences

- A preview's CPU is the one the runner chooses only while nothing else starts or stops between the preview and the apply; the start still fails with `budget.over-reservation` when the reservation no longer fits.
- A specification written for an instance the runner already holds reaches the runner only when the instance is created again, so the preview's reservation for such an instance follows the specification the plan writes, not the one the runner starts it with.
