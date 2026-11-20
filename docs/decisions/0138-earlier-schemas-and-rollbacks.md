# 0138. Earlier schemas and rollbacks

Status: accepted

## Context

Chapter 10's admission step 8 lets an update keep the running state schema, read it as an earlier schema the new state's tree accepts, or convert it through a `migrate`; `undo` of an update moves back losslessly when the schema is unchanged, when the old state type is `flexible` and its tree accepts the new one, or when the `migrate` declares an inverse, and otherwise restores the checkpoint the old bundle took. A code root records, per type, each earlier schema hash `decode_from` reads: as the type's tree accepts it, through a forward migration, or through one with an inverse. `nanos::Instance.checkpoint_as` and `Codec.encode_as` give a state under a frozen shape an inverse converts to. A flexible struct has the tree of the open struct, so only its API snapshot tells it is flexible, and decoding an encoding as such a type keeps the fields it does not know. The base bundle has no code roots: its trees are the kernel image's. The inverse of a commit is `revert`, whose change of a specification's bundle `supervisor` turns into a live update once the span commits, while an apply carries out a span's `restore-checkpoint` after its commit.

## Decision

- **Admission reads the code roots.** Each component of an admission record lists the earlier schemas its state reads, and how, and whether its state type is flexible. `migration` gives `identical` for one schema, `migrated` or `additive` when the new state reads the running schema through a migration or as its tree accepts it, and, moving to the base bundle, `additive` when its tree accepts the running one; a checkpointed base component's new state must also be accepted by the base bundle's tree.
- **The way back.** `keeps` holds for one schema, for an inverse of the running state converting to the old schema, and for a flexible old state type whose tree accepts the running one.
- **Handing under a shape.** When the bundle an instance runs declares an inverse converting to the state schema of the component it moves to, the stop of an update hands the state under that shape. A handed state the new flexible state type reads under no schema it records decodes as its own schema.
- **The undo adds the restore.** The preview of an undo finds each specification a revert moves back to another bundle; when `keeps` refuses it, it adds `restore-checkpoint <moniker> <n>` of the latest kept checkpoint that bundle took and a `dropped` record per recorded commit the instance wrote after the update, failing with `update.in-probation` while the update's probation runs and with `instance.no-checkpoint` when no such checkpoint is kept. In the effect set the update of an instance a step restores is no activation, and the restore carries the bundle the specification moves to.
- **Restore before the commit.** An apply carries out a restore carrying a bundle before its span's commit, once the read set holds: the runner reads the checkpoint and the component's description in that bundle, takes the bundle and restarts the instance from the copy, so the specification the commit writes names the bundle it already runs.

## Consequences

- An instance keeping no checkpoint of its old bundle cannot be rolled back by `undo`; it needs a plan stopping it and moving it back.
- When the span's commit fails after a restore, the instance runs the old bundle while its specification names the new one, until the next reconciling pass updates it.
- A flexible state decoding a handed encoding as its own keeps a field whose type changed as a field it does not know.
