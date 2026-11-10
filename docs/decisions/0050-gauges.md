# 0050. Gauges

Status: accepted

## Context

Chapter 13 has the runner read the counters of `/live/instances/<moniker>` and `/live/accounts/<moniker>` at request time, and a read never waits for an instance. An incarnation's cell and account are linear, so only the task holding them reads them, and drivers post to the board, so a board waiting on a driver can wait on itself. Each kernel task costs about 60 KiB of a fixed boot heap the kernel already nearly fills (ADR 0049).

## Decision

- Each incarnation's watch task holds its cell and account: it answers readings of its gauge between the handler marks it bounds, and closes them once its launch tells it the domain ended.
- The runner's `Gauges` names the watch of each running incarnation; the board reads it on a task of its own for each read of an instance's record or account, which therefore never blocks the board.

## Consequences

- An account exists while an incarnation of its instance runs.
- A read of an instance's or an account's counters costs a task while it waits for the watch.
- Wakeups, I/O bytes and energy read 0: nothing counts them per account.
