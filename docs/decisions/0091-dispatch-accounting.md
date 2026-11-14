# 0091. Dispatch accounting

Status: superseded by 0092

## Context

Chapter 08 charges a dispatch to the account its message's header carries, that of the session or instance that caused it, and a task to the account it runs under; chapter 06 has each handler call cost one timestamp read, folded into the account the header names. NaN counts the time of an instance's tasks on its executor, its handler calls included, and a header's account field was left 0. The runner's holder calls the handlers of the messages waiting in a batch, and NaN gives the header of each message it handles.

## Decision

- **Numbers.** An instance's account is numbered by its runner slot; the work of an instance carries its number, and a handler call passes on the number of the message it handles. The core's own work carries 0.
- **Folding.** The holder reads the clock around each handler call; a call of a message whose account is neither the instance's nor the core's is told to the instance's watch, which takes its time out of the instance's account and folds it into the message's account, held by number beside the gauges.
- **Reading.** An instance's gauge reads its executor's time, less what it folded out, with what was folded into its number since its incarnation started.

## Consequences

- A session's work runs on its own account, and the messages it sends carry link's number, so their handling is charged to link.
- The time of a handler call that a preemption interrupts counts in full.
