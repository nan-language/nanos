# 0018. Runner on steppers, protocols and capability kinds

Status: accepted

## Context

NaN gives a kernel what chapter 06's runner needs in these forms: a `sys::Stepper`, whose holder calls a component's handlers and starts its keyed tasks; domain runs, which a `sys::Abandoner` abandons from outside and in which a CPU exception traps with `trap.exception`; `protocol` items, whose calls `sys::Wire` encodes and serves through an outbox; capability kinds, which only their declaring module mints and reaches, never generic; and `nanos::World`, whose capability table an Env's `nanos::Environment.build` reads by key.

A request's handler answers by returning, and a call of a request exists only as an outbox call or a `sys::Wire` encoding. An outbox reaches one stepper, and an outbox post completes when a step takes the message for its handler. `sys::Cx` has no `send_to`, `reply` or `publish`, and NaN has no `nanos::Port`, `ReplyId` or `CapId` (N3, N4, N27). The base components are packages the kernel image links.

## Decision

- **Packages.** `os` holds what instances and the core share: the records of specifications, grants and lifecycles, and the kinds `Port` and `Control` with their operations. `runner` runs instances, and `supervisor` is its component. The kernel links them with its base components, its core rings and the image's genesis record.
- **Drivers.** Each instance has a driver task carrying out its lifecycle: create, start, stop, restart, lazy start, trap handling, and restarts with backoff and intensity, `supervisor` exempt from intensity. An incarnation runs in an abandoner's domain, on a cell split from the node pool at the specification's limit, or for a reserved instance from a store over the region the runner's `Reservations` give its first start and its restarts reuse, and an account split from the executor with its share. Its task builds the Env from a world holding the capabilities minted from the specification's grants and wiring, starts the stepper and calls `resume`. A watchdog per incarnation abandons the domain when a handler batch outlasts `handler_s`. The kernel's first task carves reserved instances' regions from its frame allocator for the requests the runner's tasks send it.
- **Intakes hold encoded calls.** A `Port[P]` wraps the kind `PortHandle`, whose minted port holds the target's intake, gate, protocol hash, operation mask and credit. Calls are `sys::Wire` encodings, kept by the driver across restarts and delivered one at a time. The domain serves each through `sys::Wire[P].serve`; a one-way call counts as taken once the step that took it has returned, which the holder's count of steps tells the dispatcher.
- **`Control` orders the runner.** A `Control` sends start, stop, restart and read orders, which the runner routes to drivers. The core holds one to start `supervisor`, which holds one from its genesis grant and starts the base instances in chapter 20's order.
- **The image's genesis record is code** building the base instances' specifications and grants, typed and checked by their predicates.
- **Revocation goes through the drivers.** Each driver keeps the root entries of its instance's capabilities across incarnations, each with the grant it derives from. Revoking a grant removes it and its descendants from the live grants, has every driver close the root entries it holds of them, and answers once every driver has, so every copy is stale before the revocation is acknowledged; the instance keeps running, and its next start finds the grants gone.
- **Grants expire by the RTC.** The core reads the CMOS RTC once at start, before any instance runs, and carries the reading forward by the monotonic clock, its uncertainty growing by 50 µs per second. The runner removes the grants whose expiry the latest possible wall time has reached, with their descendants, before it mints any capability, and one task of deferrable core work sleeps until the earliest expiry among live grants and revokes that grant as any revocation is; on an idle machine the expiry passes with the next wake-up.
- **Core rings** are per-CPU lists of 2,048 records of 32 bytes in the core pool, each behind a spin lock its CPU takes with interrupts disabled; a record finding its ring full is counted lost. A write posts one signal the rings share when its record enters an empty ring, fills a quarter of it or is critical. The reader keeps the 1 s timer: it waits for that signal, racing it against the clock while a ring holds a record, since a writer runs with interrupts disabled and cannot arm a deadline. An `auth.revoked` record's continuation holds its reason. Until `journal` runs, a core task is the reader and prints each event and each ring's lost count on the serial console.

Alternatives rejected:

- **Outboxes as ports:** after a restart a port would reach a stepper that has ended, and the intake could not keep its calls.
- **The runner and the kinds in the `kernel` package:** components depend on the kinds, and a package cannot depend on the program that links it; the runner's tests would also need the emulator.
- **A genesis record encoded into the image:** NaN embeds no file, and constants cannot allocate the record's maps.
- **`task::within` around the reader's wait:** once a signal's wait returns at once under it, the task's next wait traps with `trap.duplicate-access`.
- **The wall time from `clock`:** `clock` starts after the instances whose capabilities the start mints, so grants past their expiry would be minted from.

## Consequences

- Messages between instances are encoded and decoded even within the base bundle, where chapter 07 moves values.
- A handler reaches other instances only from keyed tasks, and answers a request only by returning.
- `supervisor`'s protocol has no `start`, a name NaN reserves for a component's `start` handler, and the specification's field of handler limits is `handlers`, `limits` being a NaN keyword.
- A handler batch arms one deadline.
- Grants expire by the RTC's reading alone, and a machine whose RTC gives no time expires at start every grant that has an expiry.
- Ports have no attenuation, Envs no configuration record and messages no header, failures do not escalate, only expiry revokes a grant, the core prints the events it drains instead of appending them to streams, nothing frees a removed instance's region, and no base instance the start order requires in time exists: these wait on the store, causes, nested instances, plans, `clock` and `journal`.
- `nan test` fails with `internal.invariant` when a task uses the scope it was spawned on, so only scope bodies use their scopes and each driver opens its own.
