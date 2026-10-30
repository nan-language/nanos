# 0004. Instances, capabilities and one store

Status: accepted

## Context

Every operating system until now was designed for humans at terminals. Processes, threads, users, files, paths, file descriptors, signals, pipes, environment variables, shells and init systems each answer a human need. Each also gives an agent something to parse, guess or poll: text formats, ambient authority, hidden state, configuration spread over files, no undo. NaN already provides most of what an operating system needs as language rules:

- authority as revocable capability values (NaN 14);
- domains that contain traps and settle in-flight I/O (NaN 11);
- memory budgets (NaN 12);
- structured concurrency (NaN 16);
- components, quiescent handlers whose state is typed, encodable and schema-hashed (NaN 26);
- content-addressed code with computed compatibility (NaN 29);
- `Untrusted` data (NaN 15).

## Decision

NaN OS is built from three nouns and one law:

- **Instance.** A running NaN component in its own domain is the only active thing. A program run to completion is an instance of a std adapter.
- **Capability.** The only way to reach anything. Authority persists only as grant records, from which the runner rebuilds each instance's Env at every start.
- **Store.** One versioned, transactional store holds the system definition, configuration, code, grants and data as typed values.
- **Quiescence.** Snapshots, checkpoints, previews, updates, hibernation and reverts of an instance happen only between two of its handler calls.

Every classical concept merges into these or disappears:

| Classical concept | In NaN OS |
| --- | --- |
| process | instance |
| user, group, sudo | principal and grant |
| file, directory | store node |
| pipe, socket | port |
| init, package manager | the supervisor reconciling the store |
| syslog | journal stream |
| swap | hibernation |
| shell, terminal | the link protocol (0008) |

Alternatives rejected:

- **POSIX as the base:** it brings ambient authority, text interfaces and fork.
- **Persistent capabilities:** NaN capabilities never encode, and stale authority would survive reboots.
- **Separate daemons for init, packages, cron and logging:** each is a configuration of the three nouns.
- **A model inside kernel mechanisms:** inference takes milliseconds and can be injected, where kernel decisions take microseconds. Agents improve mechanisms by writing code.

## Consequences

- The specification has one home per mechanism, and the classical utilities become requests on the link.
- Every service is introspectable, checkpointable and live-updatable by construction, since component state is Inert and Encode.
- NaN must run components and domains inside a kernel ([N2](../spec/22-nan.md#n2), [N3](../spec/22-nan.md#n3)), and an instance cannot keep linear hardware capabilities in its Env without [N28](../spec/22-nan.md#n28).
- Programs written for NaN's host targets run as jobs only when built for the NaN OS targets ([N1](../spec/22-nan.md#n1)).
