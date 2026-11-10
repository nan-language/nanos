# 0039. Device records on /live

Status: accepted

## Context

Chapter 15 has each device `devices` discovers be a `LiveDevice` record at `/live/devices/<bus path>`, each unit of a `Block` driver a record below it holding its `BlockInfo`, and each partition a record below its unit. `devices` holds those records in its state and changes them in many handlers, which cannot wait; a keyed task's stack is drawn from the component's cell when it is spawned.

## Decision

- **Capability.** `devices` publishes through a `Live` capability for `/live/devices`, which the core mints for it.
- **Batches.** Each change to a device's record marks the device. One keyed task publishes the records of the marked devices: each device's withdrawn, then its own, its units' at `u<n>` and their partitions' at `p<m>`. The next batch starts once it is done.
- **Withdrawals.** Withdrawing the record at a path withdraws those below it.

Alternatives rejected:

- **A keyed task per change:** discovery spawned dozens at once and exhausted the cell of `devices`.

## Consequences

- A device's records show it as of the last batch published.
