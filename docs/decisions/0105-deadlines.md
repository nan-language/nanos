# 0105. Deadlines

Status: accepted

## Context

Chapter 08 gives each task a deadline: its earliest enclosing `task::within`, or for a keyed task without one, its dispatch's. A message carries the deadline of the dispatch or task that sent it, and a dispatch's is its message's. An instance's next dispatch takes the earliest deadline among the messages in its intake and runs in the highest class among their accounts, in `latency` consuming the reservation of the account whose message holds that deadline. Deadlines order `latency` entities, never `normal` or `background` ones. NaN gives a task the deadline of `task::paced`, which cancels nothing, reads the calling task's deadline on a clock, and shows each task's deadline to the policy; a task it starts does not take it. The driver holds an instance's intake and delivers its calls to the domain one at a time, whose holder steps each handler call on its own task. The scheduler keeps each account's terms by executor number (ADR 0101).

## Decision

- **Stamping.** The ports the runner mints read their sending task's deadline on the runner's monotonic clock; a message takes it, else the deadline of the work it is sent from.
- **Dispatch.** The driver delivers each call with the earliest deadline among it and the calls queued behind it. The call's handler runs under a header holding that deadline, and while the call is served the holder steps under `task::paced` at it.
- **Lending.** The scheduler's terms of each account hold its number, its instance's slot or its session's, and the class of its share when its work was last made ready. Whenever the call a driver dispatches changes, it has the instance's account run in its own terms, then, while a call is dispatched, raises them by the account of each call, that of the earliest deadline first: to the highest class among them and, in `latency`, to the reservation of the first drawing on one.

Alternatives rejected:

- **Reading the deadline of the stepper's next message:** NaN gives a message's header only once its step has run.
- **Resharing the instance's account for each dispatch:** the watch holding the account would take a mark per call, and a sender's throttling would not show in the class lent.
- **Carrying the sender's class in the header:** the header's 24 bytes are the specification's, with no room for it.
- **A runner map from account numbers to executors:** each dispatch would wait on it and allocate a list of lenders, which ran the boot heap out sooner.

## Consequences

- While a call is dispatched, the instance's keyed tasks run in the terms lent too.
- A lender's class is the one the scheduler saw when its work was last made ready.
- Each dispatch costs the driver a call to the machine per account its intake holds, and one more as it ends.
