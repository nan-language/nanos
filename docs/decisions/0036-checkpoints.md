# 0036. Checkpoints

Status: accepted

## Context

Chapter 06 has the runner keep the state of a checkpointed instance as `Checkpoint` records in `/state/<moniker>/checkpoint/<n>`, keeping the last 3: one epoch after a handler call made it dirty, when a handler asks through `Cx.sync`, and when it stops dirty. A start decodes the latest checkpoint and takes an incarnation greater than the checkpoint's; `instance.started` names the checkpoint, and a failed checkpoint emits `instance.checkpoint-failed` with its code and size. Chapter 26 has a checkpoint hold the state alone, taken once the holder has drained the intake, the delayed messages and the keyed tasks, and gives `sys::Cx.sync` no messages. The driver assigns the incarnation before it launches the domain, and a kernel event record holds 8 bytes of payload and its continuation 32 more. Chapter 20 starts `store` after `supervisor`, `devices` and `fat`.

## Decision

- **Restoring.** Before drawing the cell of a checkpointed instance, its driver reads the latest checkpoint through the instance's View of `/state`. It gives the instance an incarnation past the checkpoint's and hands the checkpoint to the domain, which decodes the state under the component's schema and resumes it with `Restored`. A checkpoint the driver cannot read sends the instance to `failed`, as one the domain cannot decode does.
- **Keeping.** The holder dates the next checkpoint an epoch after the start and after the first handler call since the last checkpoint, at once when a handler calls `sync`, and keeps it once the stepper is idle. It commits the state's encoding through the View as the next record of a numbered dir keeping 3. A failure leaves the instance dirty, the next checkpoint an epoch later.
- **Stopping.** After its last handler calls, the holder checkpoints a component that is dirty before ending it.
- **Events.** The domain reports the number it restored and each failed checkpoint to the driver. The continuation of `instance.started` holds that number, and that of `instance.checkpoint-failed` the code, its argument being the size of the encoding.

Alternatives rejected:

- **Reading the checkpoint in the domain:** the incarnation the driver gave could fall below the checkpoint's.
- **Checkpointing while keyed tasks run:** the restored state would wait for what those tasks no longer post.
- **Starting fresh when `/state` cannot be read:** the instance's next checkpoints would supersede its last state.

## Consequences

- A checkpointed instance starting while `store` does not run fails, so instances before `store` in the start order are volatile.
- The epoch is 1 s by default, whatever the machine's power and storage.
- An instance whose keyed tasks never end is checkpointed only when it stops.
- `sync` appends no message once its checkpoint is durable or failed, and fails for a volatile instance.
- `restore-checkpoint`, the live update's kept checkpoint and the 24 h `/state` of a removed specification are not built.
