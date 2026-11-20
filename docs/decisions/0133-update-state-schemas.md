# 0133. Update state schemas

Status: superseded by 0138

## Context

Chapter 10's admission step 8 lets an update keep the running state schema, take one whose tree accepts it, or take one a `migrate` covers, the step's migration class showing in the effect set and in `update.started`. An instance's bundle changes through an `update` step, but also through any write of its specification, a revert and an undo; the update of a stopped instance activates nothing. A code root records each component state's schema tree and, per type, the frozen shapes its forward migrations convert from; `decode_from` also reads the earlier schemas the package's history records with a tree the type's accepts, which no code root records. The base bundle has no bundle blob: its trees are the kernel image's. A checkpointed base instance is restored by the base bundle at recovery.

## Decision

- **Admission judges.** `admission` serves `migration`: `identical` when the admission records of both bundles give the state one schema; otherwise `migrated` when a forward migration of the new state converts from the running schema, else `additive` when the new state tree accepts the running one, either only when, for a checkpointed instance of a base component, the base bundle's tree accepts the new one; else `code.schema-incompatible`. It reads a bundle's tree from the code root of the state's package in its bundle blob, and the base bundle's from the trees the kernel image gives it.
- **Migrations in the record.** Each component of an admission record lists the frozen shapes its state's forward migrations convert from, with whether an inverse converts back.
- **Every update.** The effect set has `admission` judge each update activation, whatever step makes it; an `update` step of a stopped instance has it judge the step.
- **Class at the runner.** The runner gives `update.started` the class `identical` when the state its old code hands has the new component's schema, `migrated` when a migration of the new component converts from it, else `additive`.

## Consequences

- A preview of an update between two state schemas reads both bundle blobs.
- An update judged `additive` whose new bundle's history does not record the running schema fails to decode it and reverts with `update.decode-failed`.
- Only the running generation's base bundle is consulted for a base instance.
