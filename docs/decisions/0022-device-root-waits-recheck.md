# 0022. Device root waits recheck

Status: accepted

## Context

On NaN v0.6.0, a task in an instance's domain waiting on a channel the core's keeper answers can miss the wake-up of an answer that has arrived: in the sandbox, `devices` stopped at one `declare` of its discovery on every boot, with one CPU or four. A timer armed around the wait makes it end, and a fresh wait on the same receiver takes the queued answer. The defect is in NaN's runtime, outside NaN OS.

## Decision

Every device root order that has an outcome waits for it in rounds of one second on the root's clock, which the core gives the root when it mints it. A round that ends without the outcome starts the wait again.

## Consequences

- An answer whose wake-up is lost arrives up to a second late instead of never.
- Waits on other channels, such as `Control`'s and ports', have no such guard and can still stall if the defect reaches them.
- The rounds go once NaN's runtime delivers every wake-up.
