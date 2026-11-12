# 0066. Changes through the link

Status: accepted

## Context

Chapter 14 changes the machine only through plans: `plan` previews steps on a preview branch and prints their effect set, `apply` commits it span by span exactly as previewed, `undo` previews the inverse, `confirm` keeps a provisional commit, and `put` uploads blobs; mutating requests carry request keys. The store stamps a commit's principal and session from the View it comes through, and `link` reads the tree through one View of `/` acting for `system`. The supervisor brings instances to `/sys/instances` and `/sys/config` from its subscriptions, and names in `/live/instances` the commit of the desired value it brought each instance to. The canonical rendering of a record names identifiers, hashes and commits by their text forms, which NaN's construction and JSON decoders do not read. Protocol schemas carry neither the declared class of an operation nor the names of the types they encode, and a stepper does not show the messages and keyed tasks a handler leaves held.

## Decision

- **Acting Views.** The store mints, for a View that owns the nodes their owning operations write, a View of its subtree, branch and rights acting for a session's principal in that session, keeping the root entries of the 64 latest. Previews, span commits, confirmations and uploads go through it, so history names their principal and session and the store keeps their request keys per principal.
- **Previews.** Each preview forks `preview/<n>` from the head of `main` through the store, commits each tree step on it, computes the delta from the diff of each step's commit, and deletes the branch at its end. The first read of each node is the plan's read set; each span keeps the writes the preview made and commits them with that read set, so the store's serializable reads refuse a plan whose reads changed.
- **Values.** A written value decodes by the type its node declares, its dir declares for its children, or its construction literal names, through one table of the record types the link holds, used for rendering too; a field write replaces the field in the record's JSON.
- **Activations and health.** A configuration record's change restarts its instance, as the supervisor does, and a specification's start or stop starts or stops it. An apply waits for each activated instance's status to name the span's commit, then watches its state and restarts through the health window; a provisional span waits for its record at `/sys/plans` to go. Revert by blame is the store's revert, whose commits now reach subscriptions so that the supervisor reconciles.
- **Plans kept.** The link keeps each plan by its hash for `PlanConfig.keep_s`, announces each state through the journal and `/live/plans`, and reverts at its start each span commit a restart left awaiting its confirmation.
- **Request keys and uploads.** The link keeps the result of each mutating request under its key per session. A `put`'s data lines decode in the line gatherer and reach the `put`'s keyed task through a channel the Env holds, which appends them to a store draft.

Alternatives rejected:

- **Commits through the link's own View:** history and request keys would name `system` for every plan.
- **Field writes on the canonical rendering:** the decoders do not read identifiers in their text forms.
- **Writing spans again at apply:** the preview's writes are exact once its reads hold.

## Consequences

- `call`, `redeliver` and `restore-checkpoint` steps, forked runs, deliveries and the outbox wait for operation classes and typed schemas in protocol schemas, held outputs a stepper shows, and a recording clock.
- A plan needing an approval ends its `apply` with `policy.approval-required` until approvals exist; timed plans wait too.
- Values of records holding identifiers are written as construction literals naming each identifier's fields, or as JSON.
- A specification's changes other than `stopped` reach no running instance, and an activation is no forked run.
