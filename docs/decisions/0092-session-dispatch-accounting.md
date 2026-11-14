# 0092. Session dispatch accounting

Status: accepted

## Context

Chapter 08 charges a dispatch to the account its message's header carries, that of the session or instance that caused it, and a task to the account it runs under; chapter 06 has each handler call cost one timestamp read, folded into the account the header names. NaN counts the time of an instance's tasks on its executor, its handler calls included; the runner's holder calls the handlers of the messages waiting in a batch, and NaN gives the header of each message it handles. `link` runs the work of a session's request as a domain on the session's account, from a keyed task whose header is `link`'s.

## Decision

- **Numbers.** An instance's account is numbered by its runner slot, a session's by its session number with bit 31 set, and the core's own work carries 0. The work of an instance carries its number; `link` runs the work of a session's request under its task's header with the session's number as its account; a handler call passes on the number of the message it handles.
- **Folding.** The holder reads the clock around each handler call; a call of a message whose account is neither the instance's nor the core's is told to the instance's watch, which takes its time out of the instance's account and folds it into the message's account, held by number beside the gauges. A session's account takes folds from its opening to its closing.
- **Reading.** An instance's gauge reads its executor's time, less what it folded out, with what was folded into its number since its incarnation started; a session's gauge, its account's time with what was folded into its number.

## Consequences

- The time of a handler call that a preemption interrupts counts in full.
- A handler call of a session's message that ends after the session closed is charged to no account.
