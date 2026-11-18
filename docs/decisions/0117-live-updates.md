# 0117. Live updates

Status: accepted

## Context

Chapter 10's plan step `update <moniker> <hash>` writes an instance specification's `bundle`, and reconciliation moves the running instance to the new code: it pauses the intake, has the old code encode the state, decodes that encoding with the new code, resumes it with reason `Updated`, and reverts to the old bundle from the same encoding when the new code fails during probation. The runner restarts an instance whose specification changes by stopping its incarnation and starting another; an incarnation's domain ends after handing its driver what it reports. A component of the kernel image and a held component of a bundle both encode their state on request, and both resume from a checkpoint's encoding. The driver task's stack bound leaves almost no room on the path of a start.

## Decision

- **The step.** `update` reads the instance's specification, requires its bundle admitted with a component of the specification's state type, and, when an admission record describes the component the instance runs, the same protocol hash (`port.incompatible`) and state schema (`code.schema-incompatible`); it writes the specification with the new code hash. Its class is `restorable`, and its activation an update of migration class `identical`.
- **Hand-off.** A driver whose specification names another bundle reads the component's description on a task of its own, then stops the running incarnation asking it to hand over its state: the holder encodes it before it ends, and the driver keeps a copy. A component serving another protocol leaves the instance on its old bundle.
- **Restart from the encoding.** The next start restores the copy as a checkpoint and resumes the component with reason `Updated`, starting from `StartedFrom.Update`; the queued calls wait in the driver as for any restart.
- **Revert.** A start of the new code that ends without an Env, unhealthy or unable to decode the encoding, or a trap of its domain before probation ends, puts the old bundle's description back and restarts it from the same copy, which the update then drops. Probation ends 60 s after the update's start.

## Consequences

- An update pauses the instance for a stop and a start, its keyed tasks cancelled with the stop.
- Only identical state schemas update; additive and migrated classes are refused until admission reads schema trees and `migrate`.
- Probation ends on time alone: the message count, the latency regression, the periodic health probes, the recording of inputs and their replay at a revert, the `update.*` events and `supervisor`'s revert commit are not done, so a reverted instance runs its old bundle while its specification names the new one.
