# 0008. One link protocol instead of a shell

Status: accepted

## Context

Shells, terminals and command-line tools give agents what they fail on:

- quoting and text parsing;
- interactive prompts;
- unbounded output;
- unstructured errors;
- polling;
- ambient authority.

NaN's toolchain already defines an output contract that agents use reliably: verdict lines, records, budgets with elisions naming their continuation, envelopes around foreign text, and exact fixes. An agent has no training on NaN OS, so the system must teach itself, and every token it prints costs the agent context.

## Decision

- One protocol serves every principal over every transport: serial, vsock, and TCP with Noise_IK.
- Responses follow NaN's output contract, with a default budget of 4,000 units.
- Twenty verbs cover sessions, perception, action and oversight. Everything specific to an instance is a `call` of its protocol.
- Values are read and written as NaN construction text, the notation `describe` and `Render` print, with JSON as an alternative.
- Long work answers `pending`, and its end is an event. `watch` pushes events, and nothing polls.
- `primer` prints the system's self-description, which ships with the generation and so matches the running system.
- Errors carry exact fixes that the link has dry-run before offering them.
- Rendering is canonical and stably ordered, so prompt caches hit, and perception returns deltas against a cursor.

Alternatives rejected:

- **A shell with structured output:** quoting and ambient authority remain.
- **JSON-RPC or MCP as the native wire:** more tokens and looser schemas; an adapter can map them.
- **Natural-language commands:** an interpreter of text is an injection surface, and language belongs to the agent.
- **Executables per utility:** each would need its own argument syntax and output format.

## Consequences

- The classical utilities become requests, such as `get /live/instances` in place of `ps`.
- The sandbox's link is its serial console on standard input and output, which NaN's `nan run` provides in practice but does not specify ([N22](../spec/22-nan.md#n22)).
- Decoding construction text needs [N19](../spec/22-nan.md#n19).
