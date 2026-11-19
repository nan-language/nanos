# 0124. Probation

Status: accepted

## Context

Chapter 10's plan step `update <moniker> <hash>` writes an instance specification's `bundle`, and reconciliation moves the running instance to the new code from the old code's encoding of its state, under a probation that ends on time and messages, asks `Health` periodically, compares handler latency with the previous bundle's, reverts on a trap, a failed health check or a regression, and reverts every update of the commit together; `supervisor` runs probations and commits the specification back after a revert outside a plan's health window. The runner holds each incarnation's gauge, with its message count and handler latency histogram since its start, and drives the holder, which alone may call a component between handler calls. A runner order reaches one instance's driver. An event record carries 72 bytes of payload.

## Decision

- **The step.** `update` requires the bundle admitted with a component of the specification's state type and, when an admission record describes the component the instance runs, the same protocol hash (`port.incompatible`) and state schema (`code.schema-incompatible`); it is refused while the instance's previous update is in probation (`update.in-probation`). Its activation is an update of migration class `identical`.
- **Hand-off and restart.** A driver whose specification names another bundle reads the component's description on a task of its own, stops the incarnation asking it to hand over its encoded state, and starts the new code from a copy with reason `Updated`. A bundle without the component, or with another protocol, leaves the instance on its old bundle as a reverted update.
- **The runner runs probation.** Once the new code runs, a task ticks every second: the driver delivers a `Health` holder order every `health_s`, reads the incarnation's gauge, and ends probation once `min_s` and `min_messages` or `max_s` are reached. The previous bundle's p99 is read from the old incarnation's gauge as the update stops it, over that incarnation's life, when it handled `min_messages`.
- **Reverts.** A trap, a start that fails, an `Unhealthy` reply or a regression at the end puts the old bundle's description back and restarts it from the pre-update encoding. `update.started`, `update.passed` and `update.reverted` are recorded with both code hashes, the class, the counts and the reason as codes.
- **`supervisor` follows.** A reconciling pass that finds an instance running another bundle than its specification names waits on the runner's `Probated` order, sending the bounds of `SupervisorConfig.probation` at `/sys/config/supervisor`. When the update reverts, it has the runner `Revert` the other updates of the same commit and commits the specification back, expecting the version it read.
- **Health windows.** A plan's window fails when an updated instance's live bundle changes, so a revert inside it is reverted by blame.

## Consequences

- An update pauses the instance for a stop and a start, its keyed tasks cancelled with the stop.
- The latency baseline covers the previous incarnation's life rather than its last hour.
- A hosted instance under an update's probation waits for messages a second at a time and is not suspended.
- A revert restarts the old code from the pre-update encoding without replaying the inputs probation recorded.
- Only identical state schemas update.
