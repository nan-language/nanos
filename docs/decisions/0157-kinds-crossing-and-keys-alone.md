# 0157. Env kinds crossing into isolated spaces, and keys alone in its own

Status: accepted

## Context

An instance in an isolated space reaches its Env's capabilities only when their kind implements NaN's `sys::Crossing`: the space holds a capability of the same type, whose operations, `invoke` and `call` on bytes, the kernel carries out on its own capability, `invoke` at once and `call` on a task of its own. There `sys::reach` gives `None`, `sys::derive` fails, and no capability travels in a crossing operation's arguments. NaN OS's ports, Views, sources and `Live` capabilities keep their state in the kernel: a port's route and credit, a View's scope behind the store's port, a source's journal port, the board of `/live`. A View's and a source's operations are calls of a protocol that carry the capability itself as their one capability. A bridge between two spaces carries only the calls of runner ports (nan-language/nan#298). Chapter 09 runs `keys` alone in the isolated space `keys`, and in the shared space on a machine without `user-mode`. The component entries of a nanos program are types of its root module.

## Decision

- **Kinds cross by relay.** Each operation of a port, View, source or `Live` capability first reaches its state; where that gives nothing, it has the kernel carry it out on the kernel's capability: a port's call as its protocol hash and bytes, a View's or a source's call as its bytes, which the kernel's capability sends again with itself as the call's capability, and `Live`'s reads and posts as their arguments. The kernel answers the reply's bytes, or why the call failed. A call or a reply carrying another capability fails as malformed, and the operations deriving a View fail.
- **The keyring program.** The `keys` component lives in the `keyring` nanos program, which the generation ships beside the kernel image as `keyring.nbun` and the loader passes the kernel as a module. The `keys` library keeps the component's protocol, its Env, which it builds from the world, and the work behind its operations. What the kernel knows of a start, the owner key, whether checkpoints are kept and whether `keys` runs isolated, reaches it through a `keys::Start` capability.
- **Alone in its space.** The genesis spec of `keys` asks isolation, and the runner places a base instance whose spec asks it alone in a space named after it. The core holds `keys` there through `hold_isolated`, as it holds a bundle's instance, and through `hold` in the shared space on a machine without `user-mode`. The calls of the client protocol `Sealing` reach it as calls of `Keys`, whose schema serves them.

Alternatives rejected:

- **The kernel image keeping the component, the program wrapping its state:** one instance would have two state types, each reading only its own checkpoints.
- **Ports calling through runner port numbers, to ride bridges:** components call their ports from any task, holding no `sys::Cx`.

## Consequences

- A call between instances of two isolated spaces passes through the kernel twice (N64).
- An isolated instance's calls carry no capabilities.
- `keys` runs only from the `keyring` bundle: a generation without it starts no `keys`.
