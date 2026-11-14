# 0101. Account terms

Status: accepted

## Context

Chapter 08 runs an account's `latency` work only on its reservation's CPU, has the entities of a principal's sessions divide their grant's weight on each CPU, and publishes each account's wakeups in `/live/accounts` and in metric samples. NaN's policy hooks see each task's executor number, the `id` of the account owning that executor, and run with interrupts disabled and without allocating. An account's number is distinct only while the account is open. The runner, which targets `any`, opens and closes the accounts; the scheduler's state lives in the kernel.

## Decision

- **Terms.** The scheduler's state keeps the terms of each account the runner binds, by executor number: the reservation its `latency` work draws on with that reservation's CPU, the group its entities share their weight in, and the times its work was made ready, counted at each placement.
- **Binding.** `runner::Machine` has `bind` and `unbind`. The runner binds an incarnation's account as its domain launches and again when its changed share's reservation is admitted, and a session's account as it opens and whenever its grant or the grants' reservations change. It unbinds an account before the account closes: the driver once the domain has ended, before ending the watch that closes it, and the session keeper before ending the session's gauge.
- **Tables.** The terms, and each CPU's reservations, are lists behind their spin locks. A write finding no free slot allocates a list twice as long outside the lock and swaps it in, so hooks never allocate and no count is capped.
- **Wakeups.** An account's gauge carries its executor number, by which `/live/accounts` and the metrics sampler read its wakeups from the scheduler's state.

Alternatives rejected:

- **Encoding the terms in the share's fields:** NaN interprets none of them, but they hold no room for a group or for counting wakeups, and would mean two things.
- **Fixed-size tables:** an account or reservation past their size would silently lose its terms.
- **Unbinding in the watch that closes the account:** a task the runner spawns on its drivers' scope cannot borrow the machine.

## Consequences

- Every placement and every switch takes the terms' lock, shared by all CPUs, and scans the bound accounts.
- An account's wakeups count from its binding, so an instance's restart from 0 with each incarnation.
