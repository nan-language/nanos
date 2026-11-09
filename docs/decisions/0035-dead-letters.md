# 0035. Dead letters

Status: accepted

## Context

Chapter 06 has the runner record the message a handler call was handling when its domain was abandoned as a dead letter at `/state/<moniker>/dead/<seq>`, keeping the last 64, written as the instance's principal. The runner delivers each call to the domain as its encoding, which the domain consumes, and draws on the core's own allocator. Calls carry no header and name no sender.

## Decision

- The driver keeps a copy of the encoding of the call delivered to the running domain while its handler call runs, when the encoding holds at most 16 KiB.
- When the domain is abandoned while a handler call runs, the driver writes the dead letter from a task of its scope through the instance's View of `/state`, minted at the first use from the store's endpoint for the instance's principal and moniker, with the right to write the nodes owning operations write. The write makes `dead` a numbered dir of `DeadLetter` records keeping 64, and adds the record numbered after the highest it lists.
- The dead letter names the operation by the key of the call's wire variant, and the trap by its kind and its rendering.

Alternatives rejected:

- **A copy of every call:** a block driver's 1 MiB writes would pass through the core's allocator, which they exhaust.
- **Writing from the driver's task:** the driver would wait on the store while its instance waits to restart.

## Consequences

- A dead letter of a call over 16 KiB has no message and names no operation.
- Dead letters name no sender and carry a zero header.
- Two abandonments close enough for their writes to overlap could number their dead letters alike.
