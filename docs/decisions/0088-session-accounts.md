# 0088. Session accounts

Status: accepted

## Context

Chapter 08 gives each session an account, with the share of its principal's `Session` grant, and has the accounts of a principal's sessions drain one energy bucket. Chapter 05 has a principal's sessions draw through one principal cell, of its session budget's `memory_bytes` or its pool's size, and each session through a cell split from it. `link` runs a request's work as keyed tasks, which NaN starts on the instance's own executor, and its sessions do not survive its restart, whose sessions are numbered from 1 again. NaN runs a body on a given executor and allocator as a domain whose trap ends only that domain.

## Decision

- **Capability.** `os::Accounts` has the core open, find, close and reset the accounts of sessions. The runner keeps them: a principal cell split from the pool `Machine.pool` names when a principal's first session opens and closed with its last, each session's cell split from it, each session's account split with its share, one energy bucket per principal, and each session's gauge read as `/live/accounts/s<n>`.
- **Work.** Each request's work runs as a domain on its session's account, drawing through the session's cell. A trap ends only that request: `budget.exhausted` naming `memory_bytes` when the cell ran out, NaN's trap code otherwise. An apply relays how it goes on from its domain to `link`'s outbox.
- **Lifetime.** `link` opens a session's account at its `hello` and closes it when the session ends; the kernel resets the accounts at each start of `link`.

## Consequences

- A session whose principal's sessions use up its cell fails its requests, not `link`.
- `link`'s own handling of a request, its watches and its writes to connections stay on `link`'s account.
- The entities of a principal's sessions each weigh the grant's weight until the policy can divide it (N38).
