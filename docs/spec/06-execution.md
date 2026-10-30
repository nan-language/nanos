# Execution

This chapter defines instances and their specifications, the runner, the lifecycle of an instance, restarts, durability and checkpoints, dead letters, hibernation and lazy start, forks, and jobs.

## Instances

An **instance** is a running component ([NaN 26 §Components](https://github.com/nan-language/nan/blob/main/docs/spec/26-components.md#components)), the only active thing in NaN OS. It has:

- a [moniker](02-identifiers.md#monikers);
- a [bundle](10-code.md#artefacts-and-bundles), and the component of that bundle it runs;
- its state, and its Env, which the runner builds ([Env construction](09-authority.md#env-construction));
- one intake, the FIFO of NaN 26, which receives the messages of [Communication](07-communication.md);
- exactly one domain ([NaN 11 §Domains](https://github.com/nan-language/nan/blob/main/docs/spec/11-errors-domains.md#domains)), in which its handlers and keyed tasks run (requires [N2](22-nan.md#n2), [N3](22-nan.md#n3));
- an [account](08-scheduling.md#accounting), a memory budget ([cells and limits](05-memory.md#cells-and-limits)) and a [placement](09-authority.md#placement);
- a durability class, a restart policy and a lifecycle state, defined below;
- an **incarnation**, a number the runner gives it at every start other than a revival, greater than every earlier incarnation of the instance in the current boot and than the incarnation of the checkpoint it restores.

The **supervisor** of an instance is the instance its moniker's prefix names (`app` for `app.mail`); a top-level instance's supervisor is the base instance `supervisor` ([Base instances](20-base.md)). A plan is refused (`instance.supervisor-missing`) when it writes an instance specification or runs a job whose supervisor does not exist, or removes the specification of an instance that supervises another specification. Removing an instance stops the jobs it supervises. The top-level moniker of a base instance names only that base instance: a plan that writes its specification with another `component`, or creates it on a machine without that base instance, is refused (`instance.reserved-moniker`). The moniker `auth` names no instance ([/sys/config](13-tree.md#sysconfig)): a plan that writes `/sys/instances/auth` or runs a job named `auth` is refused (`instance.reserved-moniker`).

`supervisor` reconciles `/sys/instances` with the instances that exist: as specifications change, it has the runner create, start, update, stop and remove instances. The runner carries out every transition of [Lifecycle](#lifecycle). Base instances follow the rules of this chapter; their specifications come from [genesis](13-tree.md#genesis).

### Instance specifications

`/sys/instances/<moniker>` holds the **instance specification** of every instance other than a job, an `InstanceSpec` record:

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `principal` | `Principal` | the principal of the plan writing it | the principal the instance acts for ([principals](09-authority.md#principals)) |
| `bundle` | `HashValue` | required | the admitted bundle ([admission](10-code.md#admission)) |
| `component` | `text::String` | required | the canonical path of the component state type |
| `wiring` | `mem::Map[text::String, Wire]` | empty | the ports the Env fields it names receive ([wiring](07-communication.md#wiring)) |
| `grants` | `mem::Map[text::String, u64]` | empty | the numbers of the [grants](09-authority.md#grants) whose capabilities and resources fill the Env fields it names |
| `budget` | `Budget` | [cells and limits](05-memory.md#cells-and-limits) | the memory limit, and whether a region is set aside for it ([reserved instances](05-memory.md#reserved-instances)) |
| `share` | `Share` | [shares](08-scheduling.md#shares-and-reservations) | the CPU class, weight, reservation and energy budget |
| `placement` | `auto` or `isolated` | `auto` | `isolated` runs the instance in an isolated space even when admission places its bundle in the shared space |
| `durability` | `Durability` | `volatile` | [Durability](#durability) |
| `restart` | `Restart` | `permanent` | [Restarts](#restarts) |
| `limits` | `Limits` | per field | [Handler limits](#handler-limits) |
| `lazy` | `bool` | `false` | [Lazy start](#lazy-start) |
| `stopped` | `bool` | `false` | `true` keeps the instance stopped |

The plan steps `start <moniker>` and `stop <moniker>` write `stopped`, and `update <moniker> <hash>` writes `bundle` ([steps](14-changes.md#steps)). Reconciliation applies a change to a running instance as follows:

| Change | Effect |
| --- | --- |
| `bundle` | a [live update](10-code.md#live-update) |
| `principal`, `component`, `placement`, `InstanceSpec.budget.reserved` | a restart |
| `wiring`, `grants`, or the record at `/sys/config/<moniker>` | an Env rebuild |
| `InstanceSpec.budget.limit` | applied at the next start, revival or update ([cells and limits](05-memory.md#cells-and-limits)) |
| `share`, `durability`, `restart`, `limits` | applied at the next quiescence |
| `stopped` | a start or a stop |
| `lazy` | applied at the next start |

An **Env rebuild** pauses the intake at quiescence, cancels the keyed tasks with acknowledgement ([NaN 17 §Acknowledged cancellation](https://github.com/nan-language/nan/blob/main/docs/spec/17-suspension-io.md#acknowledged-cancellation)), builds a new Env, calls `Resume` with reason `Rewired`, and resumes the intake. A **paused** intake receives messages and delivers none.

## Runner

The **runner** is the part of the core that runs instances (requires [N3](22-nan.md#n3)). It owns each instance's state, Env and intake, and takes checkpoints, hibernation images and forks. For an instance in an isolated space, it calls the handlers in that space, through that space's runtime (requires [N23](22-nan.md#n23)).

### Handler calls

The runner calls an instance's handlers, `start`, `on` and the methods of the [system interfaces](#system-interfaces), one at a time, in the instance's domain. It removes the next message from the intake only after the previous call returned. An instance is quiescent whenever none of its handlers runs ([NaN 26 §Components](https://github.com/nan-language/nan/blob/main/docs/spec/26-components.md#components)), while its keyed tasks may run. The runner reads or replaces an instance's state only at quiescence.

Each handler call costs one timestamp read, folded into the [account](08-scheduling.md#accounting) that the message's [header](07-communication.md#headers) names. An idle instance costs no timer and no wake-up.

### Held effects

Every effect a handler requests through its `sys::Cx` (`send`, `after`, `spawn`, `cancel`, and NaN OS's `send_to`, `reply`, `publish` and `sync`) and every store write it requests is held until the handler returns, then released in the order requested. A keyed task the handler spawns starts at release (requires [N8](22-nan.md#n8)). A trap discards the held effects of the handler call it interrupts. `send_to` and `reply` check their credit, port and id when requested, so their failures reach the handler at once ([sends](07-communication.md#sends)).

`Cx.publish(summary)` replaces the instance's published summary, a value of the summary type its protocol declares (requires [N3](22-nan.md#n3), [N4](22-nan.md#n4)). `/live/instances/<moniker>` serves the latest one without calling a handler ([/live](13-tree.md#live)).

### Runner tables

The runner keeps three tables per instance outside its state. Their ids are Inert, Copy and `Encode`, so the state keeps them across handler calls, hibernation and checkpoints (requires [N3](22-nan.md#n3), [N28](22-nan.md#n28)):

| Table | Id | Entry | Added when |
| --- | --- | --- | --- |
| pending replies | `nanos::ReplyId[R, E]` | a request awaiting its reply | a handler receives a request ([replies](07-communication.md#replies)) |
| received capabilities | `nanos::CapId[C]` | a capability that arrived in a message | a message carrying one is delivered ([capability transfer](07-communication.md#capability-transfer)) |
| resources | `nanos::ResId[T]` | a linear resource: a register window, a DMA allocator, a shared interrupt, a `mem::Budget` | the Env is built from a grant conferring it |

An id holds the incarnation, a slot and the slot's generation ([NaN 12 §Generational handles](https://github.com/nan-language/nan/blob/main/docs/spec/12-memory.md#generational-handles)). It is **stale** when its incarnation is not the instance's, when its slot has a newer generation, or when its entry's type differs from its type argument. Handlers use the tables through these `sys::Cx` operations:

| Operation | Arguments | Result | Failures |
| --- | --- | --- | --- |
| `reply` | `ReplyId[R, E]`, `Result[R, E]` | — | `port.stale` |
| `capability` | `CapId[C]` | a copy of the capability | `instance.stale-capability` |
| `drop_capability` | `CapId[C]` | — | — |
| `resource` | `ResId[T]` | the resource by `mut`, for the handler call | `instance.stale-resource` |
| `take_resource` | `ResId[T]` | the resource, leaving the entry vacant | `instance.stale-resource` |
| `put_resource` | `ResId[T]`, the resource | — | `instance.stale-resource` |

A keyed task uses a resource the handler took and returns it in a message; `put_resource` fills a vacant entry again. Settlement releases a taken resource with its holder ([NaN 11 §Settlement](https://github.com/nan-language/nan/blob/main/docs/spec/11-errors-domains.md#settlement)).

| Event | Pending replies | Received capabilities | Resources |
| --- | --- | --- | --- |
| hibernation, revival | kept | kept | kept |
| Env rebuild | kept | kept | kept, except those of grants no longer named, which are released |
| live update | failed with `port.server-stopped` | kept | kept |
| restart, stop, reboot | failed with `port.server-stopped` | dropped | released |

### System interfaces

A component state type may implement the std interfaces below (requires [N1](22-nan.md#n1)). They are never operations of its protocol. The runner calls each method as a handler call: it receives the state by `mut`, the Env by `mut`, a `sys::Cx[M]` and the argument listed.

| Interface | Method | Called |
| --- | --- | --- |
| `Resume` | `resume(reason)` | after `start` returns, a restore, a revival, a live update and an Env rebuild, with `reason` `Started`, `Restored`, `Revived`, `Updated` or `Rewired` |
| `Pressure` | `pressure(level)` | at the levels of [pressure](05-memory.md#pressure), with the `level` that section gives |
| `Health` | `health()`, returning `Healthy` or `Unhealthy(reason)`, `reason` a text of at most 256 bytes | every 10 s of a [health window](14-changes.md#health-windows) or a [probation](10-code.md#probation) covering the instance, from its start, and when a [driver binding](15-devices.md#driver-instances) asks |
| `Suspend` | `suspend()` and `wake()` | when [device power](15-devices.md#device-power) changes the power state of the instance's device |

`resume` re-arms what the state requires: no keyed task survives a restart, a stop, a hibernation, a live update or an Env rebuild, and pending `after` messages survive all but a restart and a stop. A trap in `health` or an `Unhealthy` reply is a failed health check; an instance whose state does not implement `Health` passes every check it does not trap in.

### Handler limits

`Limits {slow_ms: u32, handler_s: u32}` bounds handler calls:

- A handler call that returns after running 50 ms or more (`Limits.slow_ms`) emits `instance.slow-handler`.
- A handler call still running after 10 s (`Limits.handler_s`) abandons the instance's domain from outside (requires [N2](22-nan.md#n2)). The runner handles it as a trap, with `instance.handler-limit` as its kind.
- Each CPU checks its running handler's elapsed time at every timer interrupt and task switch, and its timer fires at least every 100 ms while a handler runs. The runner detects the limit within 100 ms and arms no timer per handler call.

Keyed tasks have no time limit.

## Lifecycle

An instance's lifecycle state is a `Lifecycle` value:

| State | Meaning |
| --- | --- |
| `stopped` | the instance has no domain; sends to it fail |
| `starting` | the runner builds, restores or revives it; messages queue |
| `running` | the runner calls its handlers as messages arrive |
| `hibernated` | its state is a hibernation image, or was never built; a message revives it |
| `updating` | a live update is in progress; messages queue |
| `stopping` | the runner is stopping it; sends to it fail, or queue when it restarts |
| `failed` | it ended abnormally and is not restarted |

A **stop** is a specification change setting `stopped` or removing the specification, or a cancellation of a job. A **restart** is the `restart` operation of `supervisor`, an escalation, or a change the reconciliation table maps to it.

| State | Event | Next | Effect |
| --- | --- | --- | --- |
| none | specification created | `starting`; `hibernated` when `lazy`; `stopped` when `stopped` | the intake is created |
| `stopped` | `stopped` cleared | `starting`, or `hibernated` when `lazy` | |
| `starting` | [starting](#starting) completes | `running` | `instance.started`; the intake delivers |
| `starting` | a step of starting sends it to `failed` | `failed` | `instance.failed` |
| `starting`, `running` | domain abandoned | `starting`, waiting out the backoff, or `failed` | [trap handling](#trap-handling) |
| `starting` | stop | `stopped`, or removed with its specification, while it waits out a backoff; else `stopping` once the current step of [starting](#starting) ends | step 1 of [stopping](#stopping) in a backoff, else [stopping](#stopping) |
| `starting` | restart | `starting` at once while it waits out a backoff; else `stopping` once the current step of [starting](#starting) ends, then `starting` | the intake keeps its messages |
| `running` | a job's `main` returns or fails | `stopped`, or `starting` under `permanent` | [results](#branches-and-results) |
| `running` | stop | `stopping` | [stopping](#stopping) |
| `running` | restart | `stopping`, then `starting` | the intake keeps its messages |
| `running` | update step | `updating` | [live update](10-code.md#live-update) |
| `updating` | update completed, aborted or reverted | `running` | |
| `updating` | stop or restart | `stopping` once the update completes or aborts ([live update](10-code.md#live-update)), then `starting` for a restart | [stopping](#stopping) |
| `running` | `hibernate`, or pressure on an idle checkpointed instance | `hibernated` | [hibernation](#hibernation) |
| `hibernated` | message, due `after` message, or `wake` | `starting` | revival |
| `hibernated` | stop | `stopped` | the image is dropped |
| `hibernated` | restart or update step | `starting` | revival, then the restart or update |
| `stopping` | domain freed | `stopped`, or removed with its specification | `instance.stopped` |
| `failed` | specification or configuration changed, or restart | `starting` | the restart count is reset |
| `failed` | stop | `stopped`, or removed with its specification | |
| `stopped` | specification removed | removed | |

During a live update and its probation, an abandonment reverts the update instead ([probation](10-code.md#probation)).

An instance is `converged` in `/live` when `running` or `hibernated`, or `stopped` as its specification asks or as a job that ended; `converging` in `starting`, `updating` and `stopping`; and `failed` in `failed` ([/live](13-tree.md#live)).

### Starting

1. The runner gives the instance a new incarnation, except for a revival, and draws its domain, in the space of its placement, and its budget ([cells and limits](05-memory.md#cells-and-limits)). A refused draw is handled as a trap.
2. It builds the Env from `wiring`, `grants` and the configuration record ([Env construction](09-authority.md#env-construction)). An Env field without a source, or with a source of another type, sends the instance to `failed` with `instance.env-incomplete`; a wire failing its check, with `port.incompatible`.
3. It loads the bundle when no instance uses it ([loading](10-code.md#loading)). A loading failure sends the instance to `failed` with its code.
4. A revival decodes its hibernation image, or the checkpoint standing as it ([hibernation memory](05-memory.md#hibernation-memory)); any other start of a checkpointed instance with a checkpoint decodes its latest checkpoint; any other start calls `start(mut env)`. An image or a checkpoint that does not decode sends the instance to `failed` with `instance.checkpoint-undecodable`.
5. It calls `resume`, when the state implements `Resume`.
6. It resumes the intake; the instance is `running`.

### Stopping

1. The runner stops the intake: queued one-way messages are dropped, and queued requests fail with `port.server-stopped`. For a restart, the intake keeps its messages instead.
2. It waits for the running handler call to return.
3. It checkpoints a dirty checkpointed instance.
4. It cancels the keyed tasks with acknowledgement. When they have not all ended within 5 s, it abandons the domain from outside (requires [N2](22-nan.md#n2)).
5. It fails pending requests with `port.server-stopped`, drops received capabilities, releases resources, then settles and frees the domain.

### Trap handling

When a trap, a CPU exception ([faults](04-kernel.md#faults)) or a handler limit abandons an instance's domain ([NaN 11 §Abandoning a domain](https://github.com/nan-language/nan/blob/main/docs/spec/11-errors-domains.md#abandoning-a-domain)):

1. The domain is settled, after [cut-off](15-devices.md#reset) for a driver.
2. The runner discards the interrupted handler call's held effects. When a handler call was running, its message becomes a [dead letter](#dead-letters).
3. It fails pending requests with `port.server-stopped`, or `device.reset` for a driver ([reset](15-devices.md#reset)), drops received capabilities and releases resources. The intake keeps its messages.
4. It emits `instance.trapped` with the trap kind (NaN's `trap.*`, `fault.integrity` or `instance.handler-limit`) and arms the instance's [flight recorder](17-observability.md#flight-recorders).
5. It applies the restart policy.

## Restarts

`Restart {policy, intensity: u32, period_s: u32}` decides what follows an instance's end; the runner applies it:

| Policy | Restarts after |
| --- | --- |
| `permanent` | every end that no stop requested |
| `transient` | an abandonment; never a job's return or failure |
| `temporary` | nothing |

An instance that ends by an abandonment and is not restarted enters `failed`: it drops its queued messages and fails its queued requests with `port.server-stopped`, and the runner emits `instance.failed`.

- **Backoff.** The instance restarts 10 ms × 2^n after its end, at most 30 s, where n counts its restarts since it last ran 60 s (`Restart.period_s`) without an abandonment.
- **Intensity.** An instance that has restarted 5 times (`Restart.intensity`) within the last 60 s (`Restart.period_s`) is not restarted after its next abandonment.
- **Escalation.** When a failed instance's supervisor is not `supervisor`, and the instance is not a driver ([reset](15-devices.md#reset)), its failure counts as an abandonment of that supervisor S: the descendants of S stop, deepest first and keeping their intakes, the policy of S applies to S, and the descendants start again after S.
- **`supervisor`** is restarted by the runner after every abandonment, with the backoff and without the intensity rule.

## Durability

`Durability {class, epoch_s: Option[u32], max_bytes: u64}` sets how an instance's state survives restarts:

- `volatile`: the state is never written, and every start other than a revival calls `start`.
- `checkpointed`: the runner marks the instance **dirty** whenever one of its handlers runs, and checkpoints a dirty instance one epoch after it became dirty: by default 30 s on a machine whose `power` is `Battery` or whose `storage` is `Flash` ([measured facts](01-machines.md#measured-facts)), else 1 s (`Durability.epoch_s`). An idle machine has no dirty instance and no epoch timer.

### Checkpoints

A **checkpoint** is the state's `mem::wire` encoding, taken at quiescence and committed to `/state/<moniker>/checkpoint/<n>` as a `Checkpoint` record. An encoding larger than 1 MiB (`Durability.max_bytes`) fails the checkpoint with `instance.checkpoint-too-large`, and a failed commit fails it with the commit's failure: the runner emits `instance.checkpoint-failed`, the instance is dirty, and the previous checkpoint stays the latest. A crash loses at most the state changes of one epoch. The store keeps the last 3 checkpoints of each instance.

| Field | Type | Meaning |
| --- | --- | --- |
| `number` | `u64` | increasing per instance |
| `incarnation` | `u64` | the incarnation whose state it holds |
| `bundle` | `HashValue` | the bundle that encoded it |
| `schema` | `HashValue` | the state schema |
| `time` | `WallTime` | when it was taken ([time](02-identifiers.md#time)) |
| `state` | `mem::Bytes` | the encoding |

`Cx.sync(done, failed)` takes a checkpoint when the handler returns, and appends the message `done` to the intake once its commit is durable, or `failed` when the checkpoint fails; in a volatile instance it appends `failed` at once (requires [N3](22-nan.md#n3)). The latest checkpoint at the start of a [live update](10-code.md#live-update) is kept beyond the last 3 until the update aborts or its probation ends, or, when a reboot ends that probation, until the instance's next start.

The plan step `restore-checkpoint <moniker> <n>` has the runner read checkpoint n, stop the instance, add a copy of checkpoint n numbered after the highest kept, which becomes the instance's latest checkpoint, and start the instance from that copy; it fails with `instance.no-checkpoint` when n is not kept. When a specification is removed, its `/state/<moniker>/` subtree is kept 24 h, so a specification created again within that time restores its latest checkpoint.

## Dead letters

The message a handler call was handling when its domain was abandoned is delivered again only by `redeliver`. The runner records it at `/state/<moniker>/dead/<seq>` as a `DeadLetter` record, keeping the last 64 of each instance:

| Field | Type | Meaning |
| --- | --- | --- |
| `seq` | `u64` | increasing per instance |
| `incarnation` | `u64` | the incarnation that trapped |
| `time` | `WallTime` | when it trapped |
| `header` | `Array[u8, 24]` | the message's [header](07-communication.md#headers) |
| `sender` | `Moniker` | the sending instance |
| `operation` | `text::String` | the operation's name |
| `message` | `Option[mem::Bytes]` | its `mem::wire` encoding, present when it arrived encoded or the flight recorder was armed |
| `kind` | `text::String` | the trap kind |
| `report` | `text::String` | the trap report ([NaN 11 §Trap reports](https://github.com/nan-language/nan/blob/main/docs/spec/11-errors-domains.md#trap-reports)) or the [fault](04-kernel.md#faults) report |
| `redeliveries` | `mem::List[Redelivery]` | each redelivery, `Redelivery {cause: Cause, outcome}`, its `outcome` `handled` or the `seq` of the dead letter it became |

The plan step `redeliver <moniker> <seq>` appends the message to the instance's intake under the plan's cause, with the redelivered flag of its header. It fails with `instance.no-dead-letter` when `seq` is not kept, and with `instance.dead-letter-empty` when the record has no `message`. A redelivered request has no caller, so its reply is discarded; its capability fields arrive as stale ids.

## Hibernation

An instance is **idle** when its intake is empty, no handler runs and it has no keyed task. Only checkpointed instances hibernate: memory pressure hibernates idle ones ([pressure](05-memory.md#pressure)), and the `hibernate` operation of `supervisor` any `running` one; that operation fails with `instance.volatile` or `instance.not-running` otherwise. Hibernation runs these steps:

1. The runner pauses the intake at quiescence.
2. It checkpoints a dirty state, waits until the commits of the instance's checkpoints are durable, and encodes the state into a **hibernation image**. A failed checkpoint or an encoding over `Durability.max_bytes` ends the hibernation: the intake resumes, and the instance stays `running`.
3. [Hibernation memory](05-memory.md#hibernation-memory) keeps the image, and the runner cancels the keyed tasks with acknowledgement.
4. It releases the Env and frees the domain. It keeps the intake, the runner tables, the pending `after` messages and the published summary.

A message arriving, a pending `after` message falling due, or the `wake` operation revives the instance: it runs [starting](#starting) from its image, or from the checkpoint standing as it, and keeps its incarnation.

### Lazy start

A `lazy` instance enters `hibernated` without an image when it is created or its `stopped` is cleared ([Lifecycle](#lifecycle)). Its first message starts it, not as a revival. A restart after an abandonment starts it at once.

## Forks

A **fork** is a copy of an instance that the runner makes at quiescence for a [preview](14-changes.md#preview) or a [replay](17-observability.md#replay). It encodes the state and decodes it into a new domain, under the instance's bundle or the bundle a replay names, with an Env of recording substitutes (requires [N27](22-nan.md#n27)). A fork handles only the calls the preview or replay gives it. Its held effects are recorded and never released. It never checkpoints, never appears in `/live`, and ends with its preview or replay. An instance holding a resource is never forked.

## Jobs

A **job** is an instance whose component is the std job adapter: its `resume` spawns the program's `main` as a keyed task, and the instance ends when `main` ends (requires [N1](22-nan.md#n1)). The plan step `run <job spec>` creates a job ([steps](14-changes.md#steps)). A job has no specification in `/sys/instances`, reconciliation never stops it, and it acts for the principal of the plan that ran it or, for an [extraction](19-oversight.md#extraction), of the session requesting it ([principals](09-authority.md#principals)). The runner writes its `JobSpec` to `/state/<moniker>/job` before it starts.

### Job entries

A job's bundle holds one program, whose entry has one of two forms (requires [N1](22-nan.md#n1)):

- **typed:** `fn main(take world: nanos::World, take input: I) -> O fails E uses wait`, where `I` and `O` implement `Encode` and `E` implements `Render`;
- **host-compatible:** a host entry of [NaN 27 §Entry](https://github.com/nan-language/nan/blob/main/docs/spec/27-program-model.md#entry), from a program written for host targets and built for a nanos target from source.

| `JobSpec` field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `moniker` | `Option[Moniker]` | `job-<n>`, n counting jobs on the machine | a top-level or nested moniker not in use (`instance.moniker-taken`) |
| `bundle` | `HashValue` | required | the admitted bundle |
| `input` | `mem::Bytes` | empty | the `mem::wire` encoding of `I`; one that does not decode as `I` is refused (`instance.input-mismatch`) |
| `args`, `environment` | `mem::List[mem::Bytes]`, `mem::Map[mem::Bytes, mem::Bytes]` | empty | a host-compatible entry's arguments and environment, which its `host::World` holds as `Untrusted` |
| `stdin` | `Option[HashValue]` | none | the blob the runner places at `/state/<moniker>/stdin`, which the job reads as its standard input, then end of input |
| `wiring`, `grants`, `budget`, `share`, `placement`, `limits` | as `InstanceSpec` | as `InstanceSpec` | |
| `restart` | `Restart` | `temporary` | |
| `branch` | `text::String` | `main` | the branch its writes commit to |
| `deadline_s` | `Option[u32]` | none | the job is cancelled this many seconds after its start |
| `keep_h` | `u32` | 24 | hours its `/state/<moniker>/` subtree and its branch are kept after its end |

### Worlds

The runner builds the job's World from its specification:

| Field | Capability |
| --- | --- |
| `alloc` | the job's budget |
| `exec` | the job's executor, with its share |
| `clock` | the monotonic clock, at 1 ms resolution for a bundle of provenance `agent` ([provenance](09-authority.md#provenance)) |
| `random` | the kernel's random source |
| `log` | the instance's log, written to `/journal/log` |
| `stdio` | the job's streams |
| `fs` | a `View` of the dir the grant named `fs` names, on the job's branch ([file views](11-store.md#file-views)) |
| `net` | the egress gate under the job's `Egress` grants ([egress](16-network.md#egress-gate)) |
| `caps` | `nanos::World` only: the ports of `wiring` and the capabilities of the other grants, looked up by their keys; a lookup naming no capability of the requested type fails with `instance.env-incomplete` |

A field whose grant is absent denies every operation. A host-compatible entry's `host::World` holds these fields except `caps`. A job whose bundle has provenance `agent` runs on a serial executor: one task at a time, its choices drawn from a generator seeded at its start, as on NaN's seeded scheduler ([NaN 22 §Seeded scheduler](https://github.com/nan-language/nan/blob/main/docs/spec/22-testing.md#seeded-scheduler)) (requires [N8](22-nan.md#n8), [N9](22-nan.md#n9)). Grants conferring `exec.parallel` give it the parallel executor, and `clock.fine` the nanosecond clock.

### Branches and results

1. At the job's start the runner creates the branch `job/<moniker>` at the head of `branch` ([branches](11-store.md#branches)). Every store write of the job, through `fs` or any `View` on `branch`, goes to `job/<moniker>`, and `sys::Fs` has no `sync`.
2. When `main` returns, with status 0 for a host-compatible entry, the runner has the store merge `job/<moniker>` into `branch` as one commit ([job branches](11-store.md#job-and-preview-branches)) and delete it. A failed merge or transaction commits nothing, except one that fails with `store.io` and that the next mount replays ([commit procedure](11-store.md#commit-procedure)).
3. When the job ends otherwise, nothing commits. When the job ends otherwise or its commit fails, `job/<moniker>` is kept `keep_h` hours.
4. The runner writes the `JobResult` to `/state/<moniker>/result` and emits `instance.job-ended`. A restarted job runs `main` again on a new branch.

| `JobResult` field | Type | Meaning |
| --- | --- | --- |
| `outcome` | `Returned(bytes)`, `Failed(text)`, `Trapped(text)`, `Cancelled(reason)` or `Interrupted` | the encoding of `O`, the rendering of `E`, the trap report, or what ended the job |
| `status` | `Option[u8]` | a host-compatible entry's exit status, as [NaN 27 §Host runs](https://github.com/nan-language/nan/blob/main/docs/spec/27-program-model.md#host-runs) computes it |
| `commit` | `Option[CommitId]` | the commit on `branch` |
| `commit_failure` | `Option[text::String]` | the code of a failed merge or transaction |
| `seed` | `Option[u64]` | the serial executor's seed |
| `started`, `ended` | `WallTime` | |
| `cpu_ns` | `u64` | CPU time charged to the job's account |

`Cancelled` gives its reason: `cancel`, `deadline`, `freeze` or `stop`. When the machine stops while a job runs, the runner records `Interrupted` at the next boot. After `keep_h` hours the job's moniker is free.

`/state/<moniker>/stdout` and `/state/<moniker>/stderr` are [streams](11-store.md#streams) of the bytes the job writes, one record per write, each keeping its last 16 MiB.

## Events

| Kind | Payload | Section |
| --- | --- | --- |
| `instance.started` | incarnation; `start`, `checkpoint <n>` or `revival` | [Starting](#starting) |
| `instance.stopped` | whether the domain was abandoned | [Stopping](#stopping) |
| `instance.trapped` | incarnation, trap kind, dead letter `seq` | [Trap handling](#trap-handling) |
| `instance.failed` | code or trap kind, restarts in the period, escalation target | [Restarts](#restarts) |
| `instance.hibernated` | image bytes | [Hibernation](#hibernation) |
| `instance.slow-handler` | operation, duration | [Handler limits](#handler-limits) |
| `instance.checkpoint-failed` | code, encoded bytes | [Checkpoints](#checkpoints) |
| `instance.job-ended` | outcome, commit | [Branches and results](#branches-and-results) |

## Codes

| Code | Meaning | Section |
| --- | --- | --- |
| `instance.checkpoint-too-large` | the encoded state exceeds `Durability.max_bytes` | [Checkpoints](#checkpoints) |
| `instance.checkpoint-undecodable` | a checkpoint or a hibernation image does not decode under the instance's bundle | [Starting](#starting) |
| `instance.dead-letter-empty` | the dead letter holds no message | [Dead letters](#dead-letters) |
| `instance.env-incomplete` | an Env or World field has no source, or a source of another type | [Starting](#starting) |
| `instance.handler-limit` | a handler call ran past `Limits.handler_s`, and its domain was abandoned | [Handler limits](#handler-limits) |
| `instance.input-mismatch` | a job's input does not decode as its entry's input type | [Job entries](#job-entries) |
| `instance.moniker-taken` | a job names a moniker in use | [Job entries](#job-entries) |
| `instance.no-checkpoint` | the checkpoint named is not kept | [Checkpoints](#checkpoints) |
| `instance.no-dead-letter` | the dead letter named is not kept | [Dead letters](#dead-letters) |
| `instance.not-running` | the operation needs a `running` instance | [Hibernation](#hibernation) |
| `instance.reserved-moniker` | the moniker is reserved for a base instance, or is `auth` | [Instances](#instances) |
| `instance.stale-capability` | a `CapId` is stale | [Runner tables](#runner-tables) |
| `instance.stale-resource` | a `ResId` is stale, or its entry is vacant or occupied against the operation | [Runner tables](#runner-tables) |
| `instance.supervisor-missing` | a plan leaves an instance without its supervisor | [Instances](#instances) |
| `instance.volatile` | the operation needs a checkpointed instance | [Hibernation](#hibernation) |
