# 0077. Hibernation

Status: accepted

## Context

Chapter 06 hibernates a running checkpointed instance on `supervisor`'s `hibernate`, and an idle one under memory pressure: the runner pauses its intake at quiescence, checkpoints a dirty state, encodes the state into an image, cancels the keyed tasks with acknowledgement and frees the domain, keeping the intake, the runner tables, the pending `after` messages and the published summary. A message, a due `after` message or `wake` revives it from its image, or from the checkpoint standing as it, keeping its incarnation. Chapter 05 keeps the image LZ4-compressed in the pool's hibernation cell, a sixteenth of the pool at most, while there is room. NaN's stepper gives the holder the messages its component sent itself with the time left before each enters the intake, encodes any operation of a protocol, and ends by superseding every keyed task and waiting for it; once ended, its delayed messages are released. NaN has no LZ4.

## Decision

- **The holder hibernates between handler calls.** The driver stops delivering calls and, once none is outstanding, orders the hibernation. The holder handles what waits in the intake, keeps a checkpoint when the state is dirty, encodes the state and takes the delayed messages, encoded. A failed checkpoint, an encoding over `Durability.max_bytes` or a store that keeps no checkpoints ends the hibernation, and the instance serves on. Otherwise the holder drops the keyed tasks still held, hands the driver the replies queued and the image, and ends the stepper.
- **The driver keeps what outlives the domain.** It keeps the delayed messages, ordered by the time they fall due, with one timer for the earliest, and moves each into the intake as it falls due. It keeps the requests the component postponed, which a revival answers, and numbers calls across domains. A stop, a restart or a trap drops the image and the delayed messages.
- **Images.** The driver compresses the image into the LZ4 block format through the hibernation cell, a cell of the pool limited to a sixteenth of it, or keeps it as it is when that does not shrink it. A cell without room keeps none, and the latest checkpoint stands as the image.
- **Revival.** A revival keeps the incarnation, decodes the image or else the latest checkpoint, calls `resume` with `Revived`, and records `instance.started` from `revival`, which byte 8 of its continuation marks. The hibernation records `instance.hibernated` with the bytes of the image's encoding.
- **Operations.** `hibernate` and `wake` are orders of `Control` under its `lifecycle` right. `supervisor` answers `hibernate` once the instance hibernated or serves on, or with `instance.volatile` or `instance.not-running`, and carries out `wake` like `stop` and `restart`.

Alternatives rejected:

- **Keeping the delayed messages in the image:** a checkpoint holds the state alone, so a revival from the checkpoint standing as the image would lose them.
- **Compressing in the domain:** the image would be drawn through the cell being freed.

## Consequences

- An instance hibernates only while `store` keeps its checkpoints, so host tests reach hibernation only through refusals.
- `debug` events such as `instance.hibernated` reach the journal's streams only with `JournalConfig.keep_debug`.
- Hibernation for pressure and dropping images at pressure level 3 wait for pressure levels.
