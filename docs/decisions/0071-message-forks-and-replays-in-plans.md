# 0071. Message forks and replays in plans

Status: superseded by 0075

## Context

Chapter 14 changes the machine only through plans. Besides tree steps, a plan names instances: `redeliver` delivers a dead letter again, `restore-checkpoint` restarts an instance from a checkpoint, and `call` calls an operation. A preview runs each activation and each such step's message on a fork of its instance. An apply replays each `call` and `redeliver` step on its real target with the span's targets paused within `PlanConfig.activation_ms`. The replay's store writes, held outputs, held spawns and reply must equal the fork's byte for byte, else `plan.diverged`. The span commits the replayed writes, and its irreversible held effects wait in the outbox, released only when the BLAKE3 of their payload matches; clock reads and random draws are served from the fork's recording.

NaN v0.15.0 holds every keyed task a stepper's handlers spawn until the holder starts it, giving each its class. It lets the holder take the messages the handlers sent the component and wake a waiting holder, and it encodes and serves any operation of a protocol. A port call states no class. No served clock or random source can call the one it wraps within its service's stack bound (nan-language/nan#179). Every operation of `policy` and `clock`, the components the kernel forks, reads the wall clock.

## Decision

- **Acting Views, previews, commands and revert.** The store mints, for a View owning what its owning operations write, a View acting for a session's principal in that session. Previews, span commits, confirmations and uploads go through it. A preview forks `preview/<n>` from the head of `main` and deletes it at its end. `redeliver` and `restore-checkpoint` are orders of a `Control` over every instance `link` holds. The store reverts a commit to the values of the latest retained commit of its branch's line up to its parent. Values, activations, plans kept, request keys and uploads stay as ADR 0066 decided them.
- **Message forks.** A preview runs each `call` or `redeliver` step's message at its turn among the steps, on a fork of the instance on the preview branch. The fork starts from a copy of the instance's state its holder takes between handler calls, or from the state the instance's previous fork in the plan ended with. The step takes the fork's reply and the messages its handler sent the instance, encoded with their operations' declared classes. It also takes the classes of the keyed tasks the handler spawned, joined with the operation's, and the writes the fork made to the branch, read back from its diff. The step's class joins those of its held effects.
- **Activation forks.** For each activation that starts or restarts an instance, the runner forks it fresh or from the checkpoint it resumes from, probes its health once and ends it. A trap, an `Unhealthy` reply or a fork that cannot be built is a fault of the effect set.
- **Declared steps.** A fork's clock reads the instant the fork was made and its sleeps return at once. Its random draws are zeros, and its calls to other instances are refused. A step whose fork did any of these is declared, as is a step on an instance that may hold a resource, which is never forked. A declared step takes its operation's declared class and is carried out once its span commits, through the outbox when irreversible.
- **Spans.** A span ends after a point of no return, and before a `call` or `redeliver` step to an instance an earlier step of the span activates, so that step replays on the new incarnation.
- **Pause.** A driver pauses an instance by delivering it no call. Once no call is outstanding, its holder sets aside the messages the component sent itself and handles none until it resumes, then posts them again with their remaining delays. An apply pauses the instances a span activates or messages, in moniker order within `activation_ms`, except `link` and `store`, through which it answers and commits.
- **Replay.** A holder order replays each observed step on its paused instance. Meanwhile the store sends the instance's Views of `main` to a preview branch holding the writes of the span's earlier steps. The holder posts the message under the apply's cause, handles the intake until the message is handled, and keeps the messages and keyed tasks its handler gave. The span commits the preview's writes only once each replay's reply, kept messages (by operation, class and encoding), keyed tasks' classes and writes equal its fork's.
- **Release.** Once the span commits, the instances resume and their holders release what they keep, except the irreversible messages and keyed tasks. The outbox delivers those, each released by the BLAKE3 of its payload. A failure before the commit, a revert or an instance's stop drops what the holders keep.

Alternatives rejected:

- **Replaying on a fork and adopting its state:** keyed tasks cannot move between domains, so the held spawns would be lost.
- **Committing a replay's writes as it makes them:** a divergent replay would commit part of its span.
- **Recording readings through a served clock and random source wrapping the runner's:** no service can call the one it wraps (nan-language/nan#179).

## Consequences

- Until nan-language/nan#179, recorded inputs stay empty and every step on `policy` or `clock` is declared, as their handlers read the wall clock. Replays serve steps whose handlers read neither and call no other instance.
- A replay changes its instance's state even when its span then fails; only its writes and the effects it keeps are dropped.
- A fork's store reads enter no read set: a replay reading a node changed since the preview diverges.
