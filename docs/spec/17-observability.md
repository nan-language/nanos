# Observability

This chapter defines events with their record and severities, the catalogue of event kinds, the core's per-CPU rings, the journal and its streams with their retention, causes and their propagation, traces and sampling, metrics with their names and units, flight recorders, replay, the audit chain with its entries, anchors, export and verification, and the `journal` code family.

## Events

An **event** is a typed record of something that happened, appended to the stream `/journal/events` ([streams](11-store.md#streams)). The stream entry's sequence number is the event's cursor `e<n>` ([identifiers](02-identifiers.md#identifiers)), increasing across boots, and its `time` is the wall time `journal` appended it. Its value is an `Event`:

| Field | Type | Meaning |
| --- | --- | --- |
| `time_ns` | `u64` | the monotonic time it happened, in the boot its cause names ([time](02-identifiers.md#time)) |
| `source` | `Moniker` | the instance that emitted it; for an event the core writes, the instance it concerns, else `supervisor` |
| `kind` | `text::String` | `family.name`, a kind of [event kinds](#event-kinds) or of a loaded bundle |
| `severity` | `Severity` | below |
| `cause` | `Cause` | the cause of the handler call or task that emitted it ([causes](#causes)) |
| `principal` | `Principal` | the principal of that cause, `system` for the system cause |
| `payload` | `Payload { schema: HashValue, bytes: mem::Bytes }` | the `mem::wire` encoding of the kind's payload record, with its schema hash |

`Severity` is one of `Debug`, `Info`, `Notice`, `Warn`, `Error` and `Critical`, which selectors write in lower case ([queries and selectors](18-link.md#queries-and-selectors)):

| Severity | Use |
| --- | --- |
| `debug` | detail; kept in the core rings and given to watches, committed only when `JournalConfig.keep_debug` is true (by default false) |
| `info` | a normal change of state |
| `notice` | a change a principal acts on: plan outcomes, approvals, authority and code changes |
| `warn` | degraded operation that continues |
| `error` | a failure the system contained |
| `critical` | a breach of integrity or security; `console` also shows it on the console's reserved lane ([Console](15-devices.md#console)) |

- The core and the runner write their events into the [core rings](#core-rings). An instance emits through the `emit` operation of `journal` ([the journal and its streams](#the-journal-and-its-streams)).
- The chapter defining a kind fixes its payload type. A payload renders through the emitting bundle's exported decoder (requires [N20](22-nan.md#n20)).
- A payload above 4 KiB encoded is committed as its schema, its size and its hash, without its bytes.
- The families of this specification belong to the core and the base instances: an event of such a family from another source is dropped, and `journal.kind-refused` is emitted with the source and the kind. A loaded bundle names its own families.
- `journal` takes at most 1,000 events per second from one source; beyond that it drops them and counts them in `journal.lost`, with the source and the count.

## Event kinds

Every event kind of this specification, its severity, and the section that defines it and fixes its payload:

| Kind | Severity | Home |
| --- | --- | --- |
| `boot.started` | `info` | [03](03-boot.md#events) |
| `boot.generation-good`, `boot.generation-installed`, `boot.retired`, `boot.owner-request`, `boot.loader-request` | `notice` | [03](03-boot.md#events) |
| `boot.candidate-failed` | `warn` | [03](03-boot.md#events) |
| `boot.gate-timeout` | `error` | [03](03-boot.md#events) |
| `fault.integrity` | `critical` | [04](04-kernel.md#events) |
| `fault.iommu-failed` | `error` | [04](04-kernel.md#events) |
| `fault.policy-overrun`, `fault.previous-boot-unhealthy` | `warn` | [04](04-kernel.md#events) |
| `instance.started`, `instance.stopped`, `instance.job-ended` | `info` | [06](06-execution.md#events) |
| `instance.hibernated` | `debug` | [06](06-execution.md#events) |
| `instance.slow-handler`, `instance.checkpoint-failed` | `warn` | [06](06-execution.md#events) |
| `instance.trapped`, `instance.failed` | `error` | [06](06-execution.md#events) |
| `port.undecodable` | `warn` | [07](07-communication.md#events) |
| `auth.granted`, `auth.revoked`, `auth.delegated`, `auth.key-rotated`, `auth.key-revoked`, `place.moved` | `notice` | [09](09-authority.md#events) |
| `code.admitted`, `update.started`, `update.passed` | `notice` | [10](10-code.md#events) |
| `code.loaded`, `code.unloaded`, `code.relowered` | `info` | [10](10-code.md#events) |
| `code.recheck-failed`, `update.reverted` | `error` | [10](10-code.md#events) |
| `store.mounted`, `store.scrubbed` | `info` | [12](12-store-format.md#events) |
| `store.history-thinned` | `notice` | [11](11-store.md#events) |
| `store.format-upgraded` | `notice` | [12](12-store-format.md#events) |
| `store.trial-merge-conflict` | `warn` | [11](11-store.md#events) |
| `store.writes-refused`, `store.file-lost` | `error` | [11](11-store.md#events) |
| `store.device-failed` | `error` | [12](12-store-format.md#events) |
| `store.corruption` | `critical` | [12](12-store-format.md#events) |
| `plan.previewed` | `debug` | [14](14-changes.md#events) |
| `plan.awaiting-approval`, `plan.scheduled`, `plan.applying`, `plan.health-window` | `info` | [14](14-changes.md#events) |
| `plan.applied`, `plan.denied`, `plan.expired`, `plan.delivered` | `notice` | [14](14-changes.md#events) |
| `plan.reverted` | `warn` | [14](14-changes.md#events) |
| `plan.failed` | `error` | [14](14-changes.md#events) |
| `device.added`, `device.removed`, `device.bound`, `device.unbound` | `info` | [15](15-devices.md#events) |
| `device.suspended`, `device.resumed` | `debug` | [15](15-devices.md#events) |
| `device.table-invalid`, `device.cut-off` | `warn` | [15](15-devices.md#events) |
| `device.failed`, `device.dma-fault`, `device.interrupt-storm` | `error` | [15](15-devices.md#events) |
| `net.link-up`, `net.link-down`, `net.address-added`, `net.address-removed`, `net.flow-closed` | `info` | [16](16-network.md#events) |
| `net.address-conflict`, `net.egress-denied`, `net.handshake-failed` | `warn` | [16](16-network.md#events) |
| `journal.trimmed` | `info` | [the journal](#the-journal-and-its-streams) |
| `journal.lost`, `journal.kind-refused` | `warn` | [Events](#events), [core rings](#core-rings) |
| `journal.audit-broken` | `critical` | [audit chain](#audit-chain) |
| `link.session-opened`, `link.session-detached`, `link.session-resumed`, `link.session-ended` | `info` | [18](18-link.md#events) |
| `link.hello-rejected` | `warn` | [18](18-link.md#events) |
| `policy.claimed`, `policy.recovered`, `policy.changed`, `policy.notified`, `policy.asked`, `policy.decided` | `notice` | [19](19-oversight.md#events) |
| `policy.approval-ended`, `taint.grew`, `taint.extracted` | `info` | [19](19-oversight.md#events) |
| `policy.pin-failed` | `warn` | [19](19-oversight.md#events) |
| `policy.invalid` | `error` | [19](19-oversight.md#events) |
| `policy.owner-mismatch`, `policy.frozen` | `critical` | [19](19-oversight.md#events) |

## Core rings

Each CPU has a **core ring**: one 64 KiB chunk of the core reserve ([core reserve](05-memory.md#core-reserve)) holding 2,048 records of 32 bytes. The core and the runner write a record with interrupts disabled, so each ring has one writer at a time, and `journal` is its only reader. A write takes at most the bound of [performance](21-performance.md).

| Field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| `time` | 0 | 8 | monotonic time in nanoseconds |
| `cause` | 8 | 8 | the cause, bit 63 set when traced |
| `source` | 16 | 4 | the source instance's slot in the runner's table, 0 for the core |
| `kind` | 20 | 2 | the kind's index in the kernel image's kind table |
| `severity` | 22 | 1 | 0 for `debug` up to 5 for `critical` |
| `flags` | 23 | 1 | bit 0: a continuation record follows; bit 1: a span ([traces](#traces)); other bits 0 |
| `argument` | 24 | 8 | the first 8 bytes of the payload |

A continuation record holds 32 further payload bytes; an event takes at most 3. A larger payload, a fault report among them, goes through `emit`. A record that finds its ring full is dropped and counted; `journal` reports the count as `journal.lost`, with the CPU.

`journal` drains a ring when it is a quarter full or holds a `critical` record, the writer posting its signal, or 1 s after its oldest undrained record, through a one-shot timer the core arms when a record enters an empty ring. A CPU that writes nothing arms nothing.

## The journal and its streams

`journal` is the [base instance](20-base.md#base-instances) that drains the core rings, takes events, log records and audit entries, and appends them to the streams of `/journal`, which only it writes. Its configuration record `/sys/config/journal` is a `JournalConfig`, whose fields are `keep_debug`, the five retention fields of the stream table below, `max_pct`, `metrics_h`, `trace_every`, `trace_sessions`, `trace_instances`, `recording_bytes` and `anchor_delay_s`, each with its default where this chapter uses it. It appends what it holds once the oldest record has waited 100 ms or 1 MiB is held, and at once, waiting for durability, for a `critical` event and an audit entry.

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `emit` (one-way) | `kind: text`, `severity: Severity`, `payload: Payload` | — | — | `none` |
| `audit` | `AuditEntry` without `seq` and `prev` | its `seq`, once durable | — | `none` |
| `anchor` | — | the latest `AuditAnchor`, after anchoring every entry | — | `none` |
| `record` | `moniker: Moniker`, `duration_s: u32` (at most 3,600) | — | `journal.not-forkable` | `none` |
| `replay` | `moniker: Moniker`, `seq: u64`, `bundle: Option[HashValue]`, `with_head: bool` | `ReplayReport` | `journal.not-recorded`, `journal.not-forkable`, `journal.diverged` | `none` |
| `export` | `stream: text`, `from: u64`, `to: Option[u64]` | a blob of the entries in that range | `journal.not-retained` | `none` |

Only base instances hold `audit`; operation masks keep it off every other port ([ports](07-communication.md#ports)). Every instance's `sys::Log` sends its records to `journal` ([Env construction](09-authority.md#env-construction)); a record the instance's credit cannot carry is counted in `journal.lost`.

| Stream | Entry value | Kept by default (`JournalConfig` field) | Share |
| --- | --- | --- | --- |
| `events` | `Event` | 720 h (`events_h`) | 25% |
| `log` | `LogRecord { source, level, cause, message, value, origin }` | 168 h (`log_h`) | 25% |
| `metrics` | `MetricSample`, `MachineSample` | by resolution ([metrics](#metrics)) | 15% |
| `traces` | `CauseRecord`, `Span` | 168 h (`traces_h`) | 10% |
| `faults` | `FaultRecord { moniker, incarnation, kind, report, dead_letter, cause }` | 2,160 h (`faults_h`) | 5% |
| `audit` | `AuditEntry`, `AuditAnchor` | 9,600 h (`audit_h`) | 20% |

- A `LogRecord` holds the `sys::Log` level (`info`, `warn`, `error`), the message, the rendering of the value within NaN's 1,024-byte render budget ([NaN 14 §Std capabilities](https://github.com/nan-language/nan/blob/main/docs/spec/14-authority.md#std-capabilities)), and `origin`, the label `principal.inst.<moniker>` under which the link renders its texts ([taint](19-oversight.md#taint)).
- A `FaultRecord` is named `f<n>`, n its sequence number, and is written at every trap and integrity fault ([trap handling](06-execution.md#trap-handling)).
- `/journal` holds at most 10% of the store's capacity (`JournalConfig.max_pct`), each stream the share of it the table gives; the stream's `max_age_s` and `max_bytes` follow ([streams](11-store.md#streams)). A stream that drops entries emits `journal.trimmed` with the stream and the first and last sequence numbers removed.
- `audit` drops only whole prefixes ending at an anchor, which it keeps as its first entry. Lowering `JournalConfig.audit_h` or the share of `audit` is a retention change ([presets](19-oversight.md#presets)).
- `export` with `stream` `audit` returns the entries and the anchors of the range, from the anchor preceding `from`.

## Causes

A **cause** names the request or event that started a chain of work ([identifiers](02-identifiers.md#identifiers)). The core opens causes in these cases, and `journal` appends a `CauseRecord { cause, kind, parent: Option[Cause], session: Option[SessionId], principal, subject: text, time: WallTime }` to `/journal/traces` for each:

| Opened by | Kind | Subject |
| --- | --- | --- |
| each request `link` receives | `request` | the request id; with the session and principal |
| each firing of a [timed plan](14-changes.md#timed-plans) | `schedule` | the schedule id; parent: the cause of the plan's `apply` |
| each start, restore, revival and update of an instance | `start` | the moniker and incarnation |
| each connection `net` accepts ([ingress](16-network.md#ingress)) | `connection` | the peer address |
| each boot, as its first cause `c<b>.1` | `system` | the generation |

The principal of a cause is the session's for `request`, the scheduling principal's for `schedule`, `inst.<moniker>` for `start`, `inst.net` for `connection`, and `system` for the **system cause**.

Every other piece of work carries the cause of what caused it:

- a message, a keyed task and a job, as [the message header](07-communication.md#headers) states; an `after` message, the cause of the handler call that scheduled it; a reply, the cause of its request;
- a commit, the cause of the handler call or task that requested it; a plan's commits and deliveries, the cause of its `apply` ([apply](14-changes.md#apply));
- a device's completion, the cause of the request that submitted it; its unsolicited input, the `start` cause of its driver;
- a flow's packets, the cause that opened the flow;
- the core's own work and the runner's epochs and checkpoints, the system cause.

`why <subject>` ([reading verbs](18-link.md#reading-verbs)) walks from its subject to a request, at most 32 steps:

1. A path resolves to the commit that last wrote it (role `written`), an event and a fault to their cause, and a fault also to its input (role `trapped`), whose flight-recorder header names the sender (role `sent`).
2. When the faulting instance's recorder was armed, the store reads of the handler call give the commits it read (role `read`).
3. Each cause leads to its `CauseRecord`, a `start` cause to the cause of the step or event that started the instance, and a `schedule` cause to its parent (role `scheduled`).
4. A commit or cause of a plan gives the plan with its intent (role `planned`) and its approvals (role `approved`).
5. A `request` cause gives its session, principal and request id (role `requested`) and ends the walk; so does the system cause.

A cause whose record the stream no longer keeps ends the walk with that cause.

## Traces

A **trace** is the set of spans of one cause. A cause is **traced** when its bit 63 is set in headers: `link` sets it on every 100th request cause (`JournalConfig.trace_every`, 0 for none) and on every request cause of a session that `JournalConfig.trace_sessions`, a list of sessions with expiries, names; causes opened under a traced cause are traced. A **span** is one handler call: the runner writes one to the core ring for each handler call under a traced cause, and for each handler call of an instance that `JournalConfig.trace_instances`, a list of monikers with expiries, names. Commits, device requests and flows appear as the spans of `store`, the drivers and `net`.

| Field | Type | Meaning |
| --- | --- | --- |
| `cause` | `Cause` | its cause |
| `source` | `Moniker` | the instance |
| `operation` | text | the operation handled, or the system interface method |
| `sender` | `Option[Moniker]` | the sending instance |
| `start` | `WallTime` | when the handler call began |
| `wait_ns`, `run_ns` | `u64` | time queued in the intake, and running |
| `outcome` | `returned` or `trapped(kind)` | how the call ended |

An untraced cause costs one bit test per handler call. A trace is read from `/journal/traces` by its cause ([queries and selectors](18-link.md#queries-and-selectors)).

## Metrics

`journal` samples metrics each second for every account charged anything in that second ([accounting](08-scheduling.md#accounting)); an idle account yields none. A `MetricSample` holds these fields:

| Field | Unit | Meaning |
| --- | --- | --- |
| `source` | — | the instance's moniker, or the session |
| `time`, `interval_s` | `WallTime`, s | the interval's start and length: 1, 60 or 3,600 |
| `bundle` | `HashValue` | the code hash the instance ran at the interval's end |
| `cpu_ns` | ns | CPU time charged |
| `wakeups` | count | times made runnable |
| `messages` | count | handler calls |
| `intake_max` | messages | the deepest intake |
| `memory_bytes`, `memory_peak_bytes` | bytes | the cell's charge at the end, and its peak |
| `io_read_bytes`, `io_write_bytes` | bytes | store and device bytes charged |
| `egress_bytes`, `ingress_bytes` | bytes | network bytes charged |
| `energy_mj` | mJ | estimated energy ([energy](08-scheduling.md#energy)) |
| `traps`, `restarts` | count | abandonments and restarts |
| `latency` | count per bucket | handler run times in 32 buckets: bucket 0 below 1 µs, bucket k from 2^(k−1) to 2^k µs |

A `MachineSample` is written for each second in which at least one CPU ran, with these fields:

| Field | Unit | Meaning |
| --- | --- | --- |
| `time`, `interval_s` | `WallTime`, s | as in `MetricSample` |
| `cpus` | — | one `CpuSample` per CPU, in `/live/cpus` order, holding `busy_ns` and `idle_ns` (ns) and `interrupts` and `wakeups` (count) |
| `memory` | bytes | the `MemorySummary` of `/live/memory` ([memory summary](05-memory.md#memory-summary)) |
| `commits`, `flushes` | count | the store's commits and flushes |
| `written_bytes` | bytes | the bytes the store wrote |
| `interfaces` | — | one `InterfaceSample` per interface, holding `name`, `sent_bytes` and `received_bytes` (bytes), and `sent_packets`, `received_packets`, `sent_drops` and `received_drops` (count) |

Samples of 1 s are kept 1 h, then merged into 60 s samples kept 24 h, then into 3,600 s samples kept 2,160 h (`JournalConfig.metrics_h`). A merge adds counters and histograms, keeps the last `memory_bytes` and `MachineSample.memory` and the greatest `memory_peak_bytes` and `intake_max`, and keeps the samples of different bundles apart. A percentile reads as the upper bound of the bucket holding it, as [probation](10-code.md#probation) reads p99.

## Flight recorders

The **flight recorder** of an instance always holds the headers of its last 16 inputs, 768 bytes of its runner record:

| Field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| `seq` | 0 | 8 | the input's number, increasing over the instance's incarnations in a boot |
| `time` | 8 | 8 | monotonic time at the start of its handler call |
| `cause` | 16 | 8 | its cause, bit 63 set when traced |
| `hash` | 24 | 8 | the XXH3-64 of its encoding, 0 when it moved without one |
| `sender` | 32 | 4 | the sender's slot in the runner's table |
| `size` | 36 | 4 | its encoding's bytes, 0 when it moved without one |
| `run_us` | 40 | 4 | the handler call's run time in µs, saturating |
| `operation` | 44 | 1 | the operation's number ([the message header](07-communication.md#headers)) |
| `flags` | 45 | 1 | bit 0: a request; bit 1: encoded; bit 2: redelivered; bit 3: a system interface call |
| `outcome` | 46 | 1 | 0 running, 1 returned, 2 trapped |
| reserved | 47 | 1 | 0 |

An **armed** recorder also keeps a **recording**: the state's encoding at arming, the base, then each input's encoding, moved inputs encoded on arrival, and the result of every nondeterministic call its handler calls make: clock and random reads, store reads with the commit they read, `/live` reads and the outcomes of `Cx` operations. A recorder is armed by `record` for its `duration_s`, during [probation](10-code.md#probation), and after a trap ([trap handling](06-execution.md#trap-handling)) for 10 min or until the next checkpoint.

- A recording holds at most 4 MiB (`JournalConfig.recording_bytes`), charged to the instance's cell; probation's bound is its own. Past the bound, the runner takes a new base at the next quiescence and drops the inputs before it; each checkpoint also becomes the base.
- When an armed period ends, and at a trap while armed, the runner commits the recording's `mem::wire` encoding to `/state/<moniker>/recording` as a [blob](11-store.md#nodes), replacing the previous one ([durability and checkpoints](06-execution.md#durability)).
- An instance holding a resource is never forked, so `record` of it fails with `journal.not-forkable`.

## Replay

`replay` ([`history --replay`](18-link.md#reading-verbs)) re-runs a recorded input on a fork, as the `replay` job ([jobs](20-base.md#jobs)), whose input is a `ReplayRequest { moniker: Moniker, seq: u64, bundle: Option[HashValue], with_head: bool }` holding the operation's arguments:

1. It finds the recording, in memory or in `/state/<moniker>/recording`, that holds input `seq`; none fails with `journal.not-recorded`, and an instance holding a resource with `journal.not-forkable`.
2. The runner [forks](06-execution.md#forks) the instance from the recording's base, under the recorded bundle or `bundle`; a base the bundle cannot decode fails as [starting](06-execution.md#starting) states.
3. It calls the handlers on the recorded inputs from the base up to `seq`, serving each nondeterministic call its recorded result; with `with_head`, store reads return the values of `main`'s head instead. A call the recording does not hold fails the replay with `journal.diverged`, naming the call.
4. It returns a `ReplayReport { seq: u64, bundle: HashValue, result: ReplayResult, changed: mem::List[FieldEdit], sends: mem::List[HeldOutput], replies: mem::List[mem::Bytes], writes: mem::List[Write] }`: `bundle` is the bundle that ran; `result`, one of `Returned` and `Trapped(report: text::String)`, is the outcome of input `seq`; `changed` holds the state's changed fields, before and after, as the `FieldEdit` records of [diff](11-store.md#revert-and-diff); `sends`, each a `HeldOutput { target: Moniker, operation: text::String, message: mem::Bytes }`, the encoded `replies` and the `writes` ([transactions](11-store.md#transactions)) are that input's, and none is released.

A replay counts as a preview of the requesting session ([preview](14-changes.md#preview)).

## Audit chain

The **audit chain** is the hash-chained sequence of `AuditEntry` records in `/journal/audit`. Base instances add an entry for each of these operations, through `audit`:

| `operation` | Written by | `effect` hashes |
| --- | --- | --- |
| `plan` | `link`, at each span commit and at the plan's end | the commit record, or the final `PlanEvent` |
| `delivery` | `link`, before the delivery | the [outbox](14-changes.md#the-outbox) entry |
| `extract` | `link`, before an [extraction](19-oversight.md#extraction) through a model whose `extract` operation is irreversible | the input node's hash |
| `grant`, `revoke`, `delegate` | `link`, or `supervisor` for an expiry | the grant records ([grants](09-authority.md#grants)) |
| `approval` | `policy` | the signed decision ([approvals](19-oversight.md#approvals)) |
| `admission` | `admission` | the admission record ([admission](10-code.md#admission)) |
| `policy` | `policy` | the policy record |
| `key` | `keys` | the new public key, or the revoked one |
| `freeze` | `console` or `link` | the signed request ([emergency controls](19-oversight.md#emergency-controls)) |
| `claim` | `boot` | the owner key requested ([claim](19-oversight.md#claim)) |

| Field | Type | Meaning |
| --- | --- | --- |
| `seq` | `u64` | position in the chain, from 1 |
| `prev` | `HashValue` | the hash of entry `seq − 1`, or the blob id of the genesis record for entry 1 ([genesis](13-tree.md#genesis)) |
| `time` | `WallTime` | when `journal` took it |
| `principal` | `Principal` | who acted |
| `session` | `Option[SessionId]` | the session that acted |
| `taint` | `mem::List[OriginLabel]` | the session's taint |
| `operation` | text | a value of the table above |
| `plan` | `Option[HashValue]` | the plan hash |
| `rule` | `Option[text::String]` | the policy rule that decided |
| `approval` | `Option[ApprovalId]` | the approval |
| `effect` | `HashValue` | the hash the table names |
| `result` | `ok`, `failed(code)` or `denied` | the outcome |

An entry's hash is its [hash](02-identifiers.md#hashes) of kind `audit`, over its canonical `mem::wire` encoding. An entry is durable when `audit` returns; an irreversible delivery waits for it.

An **audit anchor** is an `AuditAnchor { seq: u64, head: HashValue, time: WallTime, boot: u64, key: Array[u8, 32], signature: Signature }`: the entry it covers, that entry's hash, and an `anchor` [signature](09-authority.md#signatures) by the audit key, which `journal` obtains through its `KeyUse` capability of purpose `anchor`; the signed object, of kind `anchor`, is the anchor's encoding without `signature`. `journal` writes one 10 s after the oldest unanchored entry (`JournalConfig.anchor_delay_s`), at `anchor`, and before a power off or a reboot with unanchored entries; an idle machine writes none. After a rotation of the audit key, the old key signs the anchor covering the rotation's `key` entry before the new key signs any ([rotation and revocation](09-authority.md#rotation-and-revocation)).

A verifier, the owner's client among them, pins the audit public key when it claims the machine, and keeps its last verified anchor. It watches or exports `/journal/audit` and checks, for each new anchor:

1. the anchor's signature, by the pinned key or by a key that a `key` entry introduced under an anchor the previous key signed;
2. each entry's hash and `prev`, from its last verified anchor's head to the new anchor's head;
3. that the new anchor's `seq` is not below the last verified one and its chain extends the last verified head.

A failed check shows that the chain was rewritten or truncated. `journal` publishes two fields of `/live/security` ([/live](13-tree.md#live)): `audit_key`, the audit public key, and `audit`, `intact` or `broken`. When `journal` starts, it checks the latest anchor's signature and the chain from that anchor's head to the stream's end; a failure emits `journal.audit-broken` with the sequence number and the reason, and sets `audit` to `broken`. The `scrub` job checks the whole retained chain ([jobs](20-base.md#jobs)). While NaN OS runs, the chain is tamper-evident against code outside the shared space and `keys`. Against code that writes the volume while NaN OS is not running, it is tamper-evident only up to a verifier's last verified anchor. Code booted on the machine in NaN OS's place obtains the audit key, as does any reader of the volume on a machine without `tpm` ([keys on the machine](09-authority.md#keys-on-the-machine)).

## Codes

| Code | Meaning | Section |
| --- | --- | --- |
| `journal.not-recorded` | no recording holds the input | [Replay](#replay) |
| `journal.not-forkable` | the instance holds a resource | [Flight recorders](#flight-recorders), [Replay](#replay) |
| `journal.diverged` | the replayed handler made a call its recording does not hold | [Replay](#replay) |
| `journal.not-retained` | the range lies outside what the stream keeps | [The journal and its streams](#the-journal-and-its-streams) |
