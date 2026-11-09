# 0034. The store instance

Status: accepted

## Context

Chapter 11 has the base instance `store` serve the `Store` protocol, each operation arriving through a `View` whose subtree, branch and rights apply to it; chapter 12 places it on the `store-volume` role; chapter 13 has genesis committed into an empty store and `supervisor` reconcile `/sys/instances` and `/sys/config`. A component's state is Inert and Encode and its Env Send, while the engine holds decoders, function values that are neither. Capability kinds are handle types, which travel beside a call's encoding. On a machine target every task's stack is bounded at the final build, so a recursion, or a call through a function value whose type demands no `stack N`, fails it with `limit.stack-unbounded`; `mem::wire::Erased` demands none ([nan-language/nan#135](https://github.com/nan-language/nan/issues/135)). Moving a value out of a var's `Option` field by `match`, then assigning the var, corrupts the moved value ([nan-language/nan#134](https://github.com/nan-language/nan/issues/134)).

## Decision

- **Package.** `store` gains the module `service`: the `Store` protocol, the `View` kind, the `Service` component, its sequencer, its volume over the `Block` port and its genesis record. `runner` and `supervisor` depend on `store` for Views.
- **Sequencer.** One keyed task holds the engine for the instance's life. Handlers postpone each request and hand it, with its View's scope, to the sequencer through a bounded queue under a spin lock and a signal, the Env keeping what the queue has no room for; the sequencer answers each request through a private message and commits each run of commits through one log flush.
- **Views.** A View's state holds a port to `store`, its grant's subtree and branch, the principal it acts for and the instance writing through it. Each request carries its View as a handle, so the store reads its scope from the capability itself. Paths lie within the subtree, `/` naming its root. The runner mints Views from `Tree` grants; a View of an instance acting for `system` writes the nodes below `/sys` and `/state` that owning operations write. A commit names its View's principal and instance.
- **Typed writes.** A commit says whether derived encoders wrote its records. Decoders live in the schema registry under a function type demanding `stack 65_536`, not as `mem::wire::Erased`.
- **Bounded stacks.** The engine walks trees, dirs and wire values with explicit stacks, so no operation recurses.
- **Genesis.** The kernel builds the genesis record in code from the image's specifications and grants, with the store's retention defaults; the store commits it, with `/sys/boot` from the handoff record, when `main` is missing, and otherwise the changes of the base instances, a base moniker having one segment. It applies the retention of `/sys/config/store` at mount.
- **Subscriptions.** `subscribe` takes the subscriber's port and sends it `Watcher` messages: one per retained commit after `since` that changed the prefix, naming the prefix, then each later commit's paths that lie below it.
- **Reconciliation.** After the start order, `supervisor` subscribes to `/instances` and `/config` of its View of `/sys`. Each `changed` message runs a pass that stops or starts every instance as its specification asks, creating one the runner lacks and deleting one whose specification is gone, and restarts each instance whose configuration record the commit changed; `stop` sets `stopped` in the specification.
- **Events.** The core mints a `Reporter` through which the store reports its engine's events, which the core prints, and a failed mount, which halts the kernel with its code.

Alternatives rejected:

- **The engine in the component's state or messages:** its decoders make it neither Encode nor Send.
- **A View's scope in the call's arguments:** a caller could name any subtree.
- **A forwarding task per request:** a task spawned for every read.
- **Recursion bounded by a depth argument:** final builds judge recursion unbounded whatever its depth.

## Consequences

- With no admission records, the registry knows no kernel type, so encoded writes fail with `schema.unknown`.
- Commits name no cause, session or origin set, `store.mounted` carries the bytes replayed alone, and `LiveStore` names the role as its placement.
- A subscriber's messages wait for its port's credit, so the store sends no `Lost`.
- Instances that reconciliation creates hold no grants of their own until the runner reads `/sys/grants`.
- The sandbox's store image lies on the boot volume, whose writes do not outlast a run, so each boot formats it.
