# 0073. Outbox recovery

Status: accepted

## Context

When `link` starts again, chapter 14 has it cancel the outbox entries of each committed span still all `Held`, revert each span commit awaiting its confirmation, and deliver again the entries left `Delivering`; a keyed task's delivery fails, as its issuer's holder no longer keeps it. An outbox entry records its delivery, its target `<moniker>.<operation>`, its issuer and its payload, but neither the protocol hash of its call nor the dead letter a redelivery came from. No protocol of NaN OS declares a compensation, so `undo` reaches no delivered effect.

## Decision

- At its start, `link` first reverts the span commits awaiting their confirmation, which removes the entries those commits wrote. It then settles the outbox entries it reads from `/sys/outbox`, by plan, span and number. A span's entries all `Held` become `Cancelled`. Its `Delivering` entries are delivered again in order, and the first that fails sets the later ones `Cancelled`.
- A message an instance kept is released again while the instance still keeps it. A command is called again through the protocol and operation its target's instance describes now, with the entry's payload. A keyed task fails.
- The instances that issued the entries settled then drop what they keep.
- Each plan with entries settled fails with `plan.interrupted`, and each entry delivered again or failed emits `plan.delivered`.
- `undo` of a plan adds an `irreversible` record per delivered entry, newest first.

## Consequences

- A redelivery delivered again reaches its handler as a plain call: the instance does not flag it redelivered nor note its dead letter.
- An outbox entry's write is a commit of `link` itself, outside any plan's span.
