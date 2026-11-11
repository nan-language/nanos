# 0057. Log records

Status: accepted

## Context

Chapter 17 has every instance's `sys::Log` send its records to `journal`, which appends them to `/journal/log`, a record the instance's credit cannot carry being counted lost. NaN's `sys::LogService.log` neither fails nor waits, and its stack is bounded to 4 KiB; `task::Sender.offer` hands a value to a channel without waiting. `journal` restarts within a boot, and the core already holds the records it drains from its rings for `journal` in a backlog that its feed reads.

## Decision

- **Capability.** The runner gives each instance's world a log capability of `journal`, minted once per instance and kept across its incarnations, that offers each record to one channel of the core.
- **Losses.** A record the channel has no room for is counted on the capability, and the count travels with the instance's next record, which `journal` turns into a `journal.lost` naming the instance.
- **Backlog.** A task of the core moves the records from the channel into the backlog, which holds at most 1,024 log records and counts the rest lost; `journal` takes them with the drains of the rings and appends them with its events in one commit.
- **Record.** `journal` completes each record with its cause and its origin label `principal.inst.<moniker>`, so the capability's `log` stays within its 4 KiB.

## Consequences

- A record's cause is the boot's system cause until handler calls carry theirs.
- Records logged while `journal` is not running wait in the backlog, up to its bound.
