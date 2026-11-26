# 0162. Calls of isolated instances through their runners

Status: accepted

## Context

NaN can carry the calls an instance in an isolated space makes through a capability of a crossing kind, such as a port, over a bridge mapped into its space and the space of the instance answering them. The answering kernel admits each call through the capability's `invoke` on the call's bytes and never learns when it completes, and the call reaches the answering instance's handlers through its space's runtime. The runner owns each instance's intake: its ordering, pauses, credit leases, headers, flight recorder, and the handler calls between its checkpoints.

## Decision

The core opens no bridge. A call through a port of an isolated instance goes up its link to the core, which charges the port's credit, checks its operations and puts the call in the intake of the target's runner; the runner calls the handler in the target's space through that space's link.

Alternatives rejected:

- **Calls riding bridges, admitted by `invoke`:** the target would handle them past its runner's intake, out of reach of pauses, flight recorders and replays, and the credit a call takes at admission could not come back at its reply.

## Consequences

- A call between instances of two isolated spaces crosses the core twice, and one between an isolated and a shared instance once.
- A call or reply carrying a capability through a port of an isolated instance fails as malformed until N68.
- Supersedes 0157's consequence that awaits N64.
