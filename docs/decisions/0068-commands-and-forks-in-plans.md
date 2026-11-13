# 0068. Commands and forks in plans

Status: superseded by 0071

## Context

Chapter 14 changes the machine only through plans: `plan` previews steps on a preview branch and prints their effect set, `apply` commits it span by span exactly as previewed, `undo` previews the inverse, `confirm` keeps a provisional commit, and `put` uploads blobs; mutating requests carry request keys. Besides tree steps, a plan names instances: `redeliver` delivers a dead letter again, `restore-checkpoint` restarts an instance from a checkpoint, and `call` calls an operation. Each activation and each of these steps previews as a forked run. The runner alone holds instances, their dead letters and checkpoints in `/state`, and their states; a handler reaches the store and ports only from keyed tasks. NaN v0.13.0 gives each operation of a protocol schema its class and its types as named trees, but translates construction text through such trees with code that machine targets reject for its floats (nan-language/nan#172), and lists an operation's parameters by name rather than in declaration order (nan-language/nan#173). The store thins unrecorded commits at each index flush, so a commit's parent is often gone.

## Decision

- **Acting Views.** The store mints, for a View that owns the nodes their owning operations write, a View of its subtree, branch and rights acting for a session's principal in that session and stamping origin labels on its commits, keeping the root entries of the 64 latest. Previews, span commits, confirmations and uploads go through it with the session's taint, an upload adding `upload.<session>`.
- **Previews.** Each preview forks `preview/<n>` from the head of `main`, commits each tree step on it, computes the delta from the diff of each step's commit, and deletes the branch at its end. The first read of each node is the plan's read set; each span commits the writes the preview made with that read set.
- **Commands.** `redeliver` and `restore-checkpoint` are orders of a `Control` over every instance that `link` holds. A preview has the runner describe the instance, its protocol schema and bundle, reads the dead letter or checkpoint through the link's View, and takes the operation's declared class for a redelivery. An apply has the runner carry out each command once its span commits; an irreversible one goes through the outbox, `Held` in the span commit, then `Delivering`, `Delivered`, or `Failed` with the later ones `Cancelled`. The runner queues a redelivered dead letter itself, flagged redelivered under the plan's cause, and notes on the letter whether it was handled or became another; it restores by restarting the instance from a copy of the checkpoint added as its latest.
- **Spans.** A span ends after a point of no return, and before a redelivery to an instance an earlier step of the span activates.
- **Forks.** For each activation that starts or restarts an instance, the runner forks the instance in a domain of its own: a fresh start, or a resumption from the latest checkpoint or the one a restore names, on a world whose ports reach an intake no instance serves and whose Views and configuration record are those of the preview branch; then it probes the fork's health once and ends it. Its keyed tasks and the messages it sent itself are counted and never run. A trap, an `Unhealthy` reply or a fork that cannot be built is a fault of the effect set. The kernel names the components whose Env draws only on their world; an instance of any other component may hold a resource, is never forked, and its step's basis is `Declared`.
- **Revert.** The store reverts a commit to the values of the latest retained commit of its branch's line up to its parent: unrecorded commits change no versioned node, and a thinned recorded commit's changes are held by the retained ones after it.
- **Values, activations, plans kept, request keys and uploads** stay as ADR 0066 decided them.

Alternatives rejected:

- **Translating construction text in `link` without floats:** a second copy of std's translation, where ADR 0067 sends gaps to NaN.
- **Redeliveries through a minted port:** any holder of a port could flag a message redelivered.
- **Forking activations from the live state:** a restart resumes from the latest checkpoint, so its fork does too, without stopping the instance at quiescence.

## Consequences

- `call` steps and the `call` verb wait for nan-language/nan#172; until nan-language/nan#173, a call's unlabelled first argument binds only an operation of one parameter.
- A forked run observes faults and health, not the writes or deliveries of keyed tasks, which a fork never runs, nor the payloads of the messages an instance sends itself, which a holder cannot encode (nan-language/nan#174); `redeliver` steps are declared, nothing is replayed at apply, and recorded inputs stay empty.
- A plan needing an approval ends its `apply` with `policy.approval-required` until approvals exist, and a timed plan applies only while the `link` that scheduled it runs.
- An outbox entry left `Delivering` by a restart of `link` is not delivered again.
