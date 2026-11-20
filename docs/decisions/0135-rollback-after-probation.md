# 0135. Rollback after probation

Status: superseded by 0138

## Context

Chapter 10 has `undo` of an update commit preview an update back to the old code hash, lossless when the schema is unchanged, when the new schema is additive over the old one and the old state type is `flexible`, or when the `migrate` declares an inverse; otherwise the preview offers `restore-checkpoint` of the checkpoint the old bundle took before the update, and lists the commits the instance wrote since. The inverse of a commit is `revert`, whose change of a specification's bundle `supervisor` turns into a live update once the span commits, while an apply carries out a span's `restore-checkpoint` after its commit: a live update the old code cannot decode would run first and revert. A flexible struct has the schema tree of the open struct; only its API snapshot tells it apart. The new code of an update is a bundle's, and `nanos::Instance` gives its state under its own schema only (nan-language/nan#265).

## Decision

- **Admission judges the way back.** `admission` serves `keeps`: one schema, a forward migration of the old state from the running schema, or an old tree accepting the running one on a type the old bundle's snapshot declares `flexible`. The base bundle's types count as not flexible.
- **The undo adds the restore.** The preview of an undo finds each specification a revert moves back to another bundle; when `keeps` refuses it, it adds `restore-checkpoint <moniker> <n>` of the latest kept checkpoint that bundle took, and a `dropped` record per recorded commit the instance wrote after the update. It fails with `update.in-probation` while the update's probation runs, and with `instance.no-checkpoint` when no such checkpoint is kept.
- **The restore replaces the update.** In the effect set, the update of an instance a step of the plan restores is no activation; the restore carries the bundle the specification moves to.
- **Restore before the commit.** An apply carries out a restore carrying a bundle before its span's commit, once the read set holds: the runner reads the checkpoint and the component's description in that bundle, takes the bundle and restarts the instance from the copy, so the specification the commit writes names the bundle it already runs.

## Consequences

- An update whose `migrate` declares an inverse rolls back through a checkpoint until NaN gives a bundle component's state under an earlier shape.
- An instance keeping no checkpoint of its old bundle cannot be rolled back by `undo`; it needs a plan stopping it and moving it back.
- When the span's commit fails after the restore, the instance runs the old bundle while its specification names the new one, until the next reconciling pass updates it.
