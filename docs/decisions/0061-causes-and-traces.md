# 0061. Causes and traces

Status: accepted

## Context

Chapter 07 gives every message a 24-byte header holding its cause, deadline, account and flags, and chapter 17 has the core open a cause for each start of an instance, has every piece of work carry the cause of what caused it, names in each event the principal of its cause, and has the runner write a span to the core rings for each handler call under a traced cause or of an instance `JournalConfig.trace_instances` names. NaN gives each task a 32-byte `sys::Header` that the tasks it starts and the messages it queues carry, and `sys::run_headed` runs a body under a header; a stepper's `step` handles a message under its header without telling the runner which. Each CPU has its own ring, and `journal` drains them in the order they fall due.

## Decision

- **Headers.** A task's `sys::Header` holds the chapter 07 header, then the slot of the instance doing the work, 0 for the core. A port stamps each call with the header of the task sending it; the runner runs each incarnation's domain under its start cause and serves each call under the call's header, both naming the instance's slot, and runs its own work and the core's orders under the system cause.
- **Causes.** The core numbers causes from the system cause in each boot. Each start opens one under the cause of the order, trap or call that started the instance, written to the rings as a record flagged as an opened cause, its principal `inst.<moniker>` noted at once in a table of the 256 latest that `journal` reads through its feed for the principal of each event. Events, log records and commits take the cause of the work that made them.
- **Spans.** The driver of an instance begins a span when it delivers a call whose cause is traced or while the instance is traced, timed from the call's arrival and delivery, and writes it when the handler call ends or traps. `journal` hands `trace_instances` to the core through its feed at each resume. `/journal/traces` holds `TraceRecord`, a cause record or a span.

## Consequences

- Only calls between instances have spans: the handler calls of messages an instance sends itself, and of its keyed tasks' posts, have none until a step tells the runner its header ([N36](../spec/22-nan.md#n36)).
- A span's run time runs from the call's delivery to the driver learning that its handler call ended.
- No header holds an operation number.
- An event whose cause is older than the 256 latest opened names the principal `system`.
- Code in the shared space can run under any header through `sys::run_headed`, so causes there are only as trustworthy as that code.
