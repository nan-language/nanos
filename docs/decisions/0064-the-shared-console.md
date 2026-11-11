# 0064. The shared console

Status: accepted

## Context

Chapter 20 has `console` drive the reserved lane of the `console` role, and on a shared console `link` passes it every input line starting `console `, treating none as a request; its output goes ahead of pending link output, each a block of the id `console`. The UART driver serves the reserved lane as `write_reserved` and `read_reserved`, and the link holds only `write` and `read`. Claims, approvals, PINs and emergency controls come with the oversight task.

## Decision

- **Package.** `console` holds the `console` component, a base instance the supervisor starts at step 9 with the reserved lane of the console role and a read View of `/live`.
- **Lines.** The link's serial reader recognises a line starting `console ` outside a request, and a keyed task posts the rest of it to `console`'s `line`, the one operation the link's port to it allows.
- **Answers.** `console` answers each line from a keyed task and writes the blocks to the reserved lane one at a time: `status` prints `/live/status` and `/live/security`, and any other line fails with `request.unknown-verb`.

## Consequences

- The console reads no input itself while it is shared, so the dedicated console waits on the oversight task.
