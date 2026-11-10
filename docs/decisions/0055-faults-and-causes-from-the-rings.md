# 0055. Faults and causes from the rings

Status: accepted

## Context

Chapter 17 has `journal` write a `FaultRecord` to `/journal/faults` at every trap and integrity fault, and a `CauseRecord` to `/journal/traces` for each cause the core opens, the system cause `c<b>.1` first at each boot. The core already writes `instance.trapped`, `fault.integrity` and `boot.started` to its rings, which `journal` drains in order.

## Decision

- **Derived records.** The drain derives a fault record from each `instance.trapped` and `fault.integrity` it takes, and the system cause's record from `boot.started`, committing them with the events of the same drain.
- **Traces stream.** `/journal/traces` holds `CauseRecord`s, with its share and retention of chapter 17.
- **Wall time.** `journal` reads the time a cause opened on the wall clock of the core's start, carried to the record's monotonic time.

## Consequences

- A fault record names the trap's kind and the instance's incarnation; its report and dead letter stay empty until the core hands them to `journal`.
- Only the system cause has a record: no other cause is opened yet.
