# 0130. Update state schemas

Status: superseded by 0133

## Context

Chapter 10's admission step 8 lets an update keep the running state schema, take one whose tree accepts it, or take one a `migrate` covers, the step's migration class showing in the effect set and in `update.started`. An instance's bundle changes through an `update` step, but also through any write of its specification, a revert and an undo; the update of a stopped instance activates nothing. Code roots record each component state's schema tree but no migration, so no artefact tells which older schemas a `migrate` covers. The base bundle has no bundle blob: its trees are the kernel image's. A checkpointed base instance is restored by the base bundle at recovery.

## Decision

- **Admission judges.** `admission` serves `additive`: `identical` when the admission records of both bundles give the state one schema; otherwise `additive` when the new state tree accepts the running one and, for a checkpointed instance of a base component, the base bundle's tree accepts the new one; else `code.schema-incompatible`. It reads a bundle's tree from the code root of the state's package in its bundle blob, and the base bundle's from the trees the kernel image gives it.
- **Every update.** The effect set has `admission` judge each update activation, whatever step makes it; an `update` step of a stopped instance has it judge the step.
- **No `migrated` class.** An update only a `migrate` covers is refused.
- **Class at the runner.** The runner gives `update.started` the class `identical` when the state its old code hands has the new component's schema, else `additive`.

## Consequences

- A preview of an update between two state schemas reads both bundle blobs.
- Updates relying on a `migrate` wait for code roots recording migrations.
- Only the running generation's base bundle is consulted for a base instance.
