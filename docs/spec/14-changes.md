# Changes

This chapter defines how the machine changes: reversibility and compensations, plans and their steps and options, previews and the effect set, the plan hash, apply with its spans and the outbox, the plan lifecycle, health windows, provisional commits, undo, request keys, concurrency, timed plans, and the codes of the `plan` family.

## Reversibility

**Reversibility** is the class of an effect, named by what undoes it:

| Class | Effect | Undone by |
| --- | --- | --- |
| `none` | nothing outside the operation's own values; writes to `ephemeral` dirs, `/state` and `/journal` | nothing |
| `recorded` | a [commit](11-store.md#transactions) under `/sys` or `/data` | a [revert](11-store.md#revert-and-diff) |
| `restorable` | live state derived from the tree: running [instances](06-execution.md#instances), their bundles, wiring, [grants](09-authority.md#grants), configuration and placement, device settings; the running generation | reconciliation to the tree, which may restart instances; for the running generation, a rollback ([undo](#undo)) |
| `irreversible{kinds}` | `emit`: data leaves the machine; `actuate`: a physical effect no reconciliation restores; `erase`: information no history restores; `spend`: money, credits or a declared high-energy mode | a compensation, when one is declared |

The classes are ordered `none` < `recorded` < `restorable` < `irreversible{K}`, and `irreversible{K1}` ≤ `irreversible{K2}` when K1 ⊆ K2. The join of two classes is the greater; the join of two irreversible classes is irreversible with the union of their kinds. A class renders as its name; an irreversible one as `irreversible{`, its kinds in the order `emit`, `actuate`, `erase`, `spend` separated by `,`, then `}`.

```nan
pub closed enum Reversibility derives Copy, Eq, Encode {
    None
    Recorded
    Restorable
    Irreversible(kinds: IrreversibleKinds)
} Reversibility

pub closed struct IrreversibleKinds derives Copy, Eq, Encode {
    pub emit: bool
    pub actuate: bool
    pub erase: bool
    pub spend: bool
} IrreversibleKinds
```

### Declared classes

- Code: the reversibility field of a NaN digest ([NaN 10 §Reserved field](https://github.com/nan-language/nan/blob/main/docs/spec/10-effects-digests.md#reserved-field)) holds each function's class, joined over its calls as the other digest facts are (requires [N16](22-nan.md#n16)).
- Primitives: std fixes the class of each target-dependent operation per target. On `x86_64-nanos`, store writes are `recorded`; `Clock`, `Random` and `Log` operations are `none`; `Egress` sends are `irreversible{emit}`; a `KeyUse` erase is `irreversible{erase}`; a send through a `Port[P]` has the class P declares for its operation; a message an instance sends itself, through `Cx.send`, `Cx.after` or its keyed tasks' outboxes, has the class its own protocol declares for that message's operation (requires [N16](22-nan.md#n16)).
- Protocols: each protocol operation declares the greatest class its implementations reach; admission refuses an implementation whose digest exceeds it (requires [N4](22-nan.md#n4), [N16](22-nan.md#n16)).
- Devices: each operation of a device protocol declares its class as a premise sealed by a `vendor.*` or `human.*` [principal](09-authority.md#principals) (requires [N15](22-nan.md#n15), [N16](22-nan.md#n16)).
- Encapsulation: an operation declares a class lower than that of the operations it calls only as a premise sealed in the same way, stating that its own mechanism undoes their effects; the declaration is then a trust escape. The store's `commit` is `recorded` although it writes blocks through `Block.write`, which is `irreversible{erase}` (requires [N15](22-nan.md#n15), [N16](22-nan.md#n16)).

A declared class bounds every execution of an operation; a preview observes one execution, whose class may be lower (a store write that lands in an `ephemeral` dir is `none`). Admission records the declared class of every operation of a [bundle](10-code.md#artefacts-and-bundles) ([admission](10-code.md#admission)).

### Compensations

A **compensation** is a sealed claim that an operation B of a protocol undoes an operation A of a protocol, with a pure function from A's request and reply to B's request (requires [N16](22-nan.md#n16)). Undo calls the compensations of delivered irreversible effects ([undo](#undo)).

## Plans and steps

A **plan** is a list of steps with options, previewed together and applied by its plan hash. Steps take effect in their order. `link` previews and applies plans, `policy` decides their approvals, and `supervisor` performs their activations ([lifecycle](06-execution.md#lifecycle)).

### Steps

| Step | Class | Effect |
| --- | --- | --- |
| `write <path> <value>` | `recorded` | writes a record or blob node, creating missing parent dirs |
| `write <path>.<field> <value>` | `recorded` | writes one field of a record node, keeping its other fields |
| `delete <path>` | `recorded` | removes the node and its subtree |
| `install <path or hash>` | `recorded` | admits the [bundle](10-code.md#artefacts-and-bundles) blob at the path or with the hash and writes `/sys/code/<hash>` ([admission](10-code.md#admission)) |
| `start <moniker>`, `stop <moniker>` | `restorable` | write `InstanceSpec.stopped` ([instance specifications](06-execution.md#instance-specifications)) |
| `update <moniker> <hash>` | `restorable` | sets the [instance](06-execution.md#instances)'s bundle; its activation is a live update ([live update](10-code.md#live-update)) |
| `grant <spec>` | `recorded` | writes `/sys/grants/<id>` from a construction literal of a grant without its generated fields ([grants](09-authority.md#grants)) |
| `revoke <grant id>` | `recorded` | removes the grant and its descendants ([revocation](09-authority.md#revocation)) |
| `call <moniker> <op>(<args>)` | observed | sends a request to the instance |
| `revert <commit>` | `recorded` | writes the inverse of the commit, three-way checked against the [commits](11-store.md#transactions) after it ([revert](11-store.md#revert-and-diff)) |
| `run <spec>` | declared by the job's entry | starts a job acting for the plan's principal, from a `JobSpec` construction literal ([jobs](06-execution.md#jobs)) |
| `generation <manifest>` | `restorable` | installs the generation whose manifest blob is at the path, its kernel image and genesis record being blobs the store holds, and reboots ([generation switch](10-code.md#generation-switch)) |
| `redeliver <moniker> <seq>` | observed | sends the instance's dead letter `<seq>` again ([dead letters](06-execution.md#dead-letters)) |
| `restore-checkpoint <moniker> <n>` | `restorable` | restarts the instance from a copy of its checkpoint `<n>`, added as its latest checkpoint ([checkpoints](06-execution.md#checkpoints)) |

- Every step but `call`, `redeliver` and `restore-checkpoint` is a **tree step**: it writes the tree, and activation applies what it writes. A step whose commit changes an instance's spec, its configuration record `/sys/config/<moniker>` or its grants **activates** that instance, and its class is at least `restorable`. A `restore-checkpoint` step activates its instance. An activation of an instance in `failed` restarts it.
- A `<value>` is a construction literal of the node's declared type (requires [N20](22-nan.md#n20)), a JSON value of that type with `--json-in`, or for a blob node the id of a blob the store holds.
- In `write <path>.<field>`, the last path segment splits at the end of its longest prefix ending before a `.` that names a record node at the base; the rest is a field path (`limits.max`).
- `write` and `delete` take `--expect <commit or hash>`: the node's last writing commit or its content hash at the base must equal it (`plan.stale`); the node then joins the read set.
- The arguments of `call` follow NaN's call form, the first unlabelled and the others labelled, and decode against the operation's parameter types (requires [N20](22-nan.md#n20)).
- A `generation` step is the last step of its plan, and a plan for a [branch](11-store.md#branches) other than the running branch holds only `write`, `delete`, `install`, `grant`, `revoke` and `revert` steps (`plan.step-misplaced`).

### Options

| Option | Default | Effect |
| --- | --- | --- |
| `--intent "<text>"` | none | the [principal](09-authority.md#principals)'s stated purpose, stored in each commit's `intent` and shown to approvers in an envelope |
| `--confirm-within <s>` | 120 s when the preview finds a transport cut (`PlanConfig.confirm_s`), else none; `0` disables | makes each span's commit provisional ([provisional commits](#provisional-commits)) |
| `--health <s>` | 30 s (`PlanConfig.health_s`); `0` disables | the length of each span's health window ([health windows](#health-windows)) |
| `--at <branch>` | the running branch | the branch the plan commits to |

`PlanConfig` is the `plans` field of the link's configuration record ([configuration](18-link.md#configuration)).

## Preview

`plan` previews a plan:

1. Parse each step and check it against the [session](18-link.md#connections-and-sessions)'s [grants](09-authority.md#grants); a step outside them fails with its authority code.
2. Fork the head of the target [branch](11-store.md#branches) as the preview branch `preview/<n>` ([job and preview branches](11-store.md#job-and-preview-branches)); that head is the plan's **base**.
3. Apply each tree step to the preview branch; a value its node's type rejects fails with its `schema` code ([validity](11-store.md#declared-types-and-validity)).
4. Execute each `call` and `redeliver` step and each activation as a forked run.
5. Compute the effect set, the spans and the classes; a horizon fails the preview with `plan.horizon`.
6. Ask `policy` to decide the plan ([policy](19-oversight.md#policy)), recording the approvals it requires.
7. Compute the plan hash, keep the plan for 1 h (`PlanConfig.keep_s`) and emit `plan.previewed`.

A failure ends the preview, which commits nothing; the store deletes the preview branch when the preview ends. Previews are charged to the session's [account](08-scheduling.md#accounting); at most 4 run at once over all the sessions of its principal (`SessionBudget.previews`), and another fails with `budget.previews` ([budgets](18-link.md#budgets)).

### Forked runs

A **forked run** runs, one level deep, a `call` or `redeliver` step's message, or an activation's `start`, or `Resume` when the instance keeps its state, then one `Health` probe, on a [fork](06-execution.md#forks) of the [instance](06-execution.md#instances): a fresh one for a `Start` activation, its state decoded or migrated for the new [bundle](10-code.md#artefacts-and-bundles) for an `Update` (requires [N3](22-nan.md#n3), [N5](22-nan.md#n5)). `Stop` and `Reboot` activations run nothing.

- The fork's substitutes serve store reads and writes from the preview branch; its sends to other instances and the messages it sends itself through `Cx.send` and `Cx.after` become **held outputs**, and the keyed tasks it spawns **held spawns**, which never run; calls of class above `recorded` on its other capabilities are recorded and answered with the operation's `Unit` success, and a call whose result is not `Unit` ends the run with the fault `opaque` (requires [N27](22-nan.md#n27)).
- A trap or an `Unhealthy` reply is a fault of the effect set and ends the run.
- An instance holding a resource is never forked: a step targeting it takes its operation's declared class, and its effects are `Declared`.
- The forks' encoded states and the recorded inputs of one plan total at most 1 MiB (`PlanConfig.preview_bytes`); beyond, the preview fails with `plan.preview-too-large`.

A forked run records its nondeterministic inputs:

| Input | Recorded | At apply |
| --- | --- | --- |
| store read | the path, the [commit](11-store.md#transactions) that last wrote the node, and the value | validated, then served from the recording |
| `/live` read | the path and the hash of the value | read again and compared |
| clock read, random draw | the value | served from the recording |
| result of a recorded capability call | the value | served from the recording |

The store reads, the nodes that tree steps write and the nodes `--expect` names form the plan's **read set**.

### Horizon

The **horizon** of a plan is its earliest irreversible step S such that a step after S forks an instance S delivers to, or reads a node that an instance S delivers to may write through its `View` grants. The preview of a plan with a horizon fails with `plan.horizon`, whose fix splits the plan after S.

## The effect set

An **effect set** is what a preview computes:

```nan
pub closed struct EffectSet derives Encode {
    pub base: CommitId
    pub branch: text::String
    pub steps: mem::List[StepEffect]
    pub delta: mem::List[NodeChange]
    pub activations: mem::List[Activation]
    pub widenings: mem::List[Widening]
    pub deliveries: mem::List[Delivery]
    pub faults: mem::List[Fault]
    pub approvals: mem::List[ApprovalNeed]
    pub cuts: mem::List[TransportCut]
    /// The origin labels of the previewing session's taint.
    pub taint: mem::List[OriginLabel]
    pub costs: Costs
} EffectSet

pub closed struct StepEffect derives Encode {
    pub step: u32
    pub span: u32
    pub class: Reversibility
    pub basis: Basis
    /// The encoded reply of a `call` step.
    pub reply: Option[mem::Bytes]
} StepEffect

pub closed enum Basis derives Copy, Eq, Encode {
    Observed
    Declared
} Basis

pub closed struct NodeChange derives Encode {
    pub path: Path
    pub step: u32
    /// Content hashes; `None` where the node is absent.
    pub before: Option[HashValue]
    pub after: Option[HashValue]
} NodeChange

pub closed struct Activation derives Encode {
    pub moniker: Moniker
    pub step: u32
    pub kind: ActivationKind
    pub migration: Option[Migration]
    /// The expected intake pause.
    pub pause_ms: u32
} Activation

pub closed enum ActivationKind derives Copy, Eq, Encode {
    Start
    Stop
    Restart
    Update
    Rebuild
    Reboot
} ActivationKind

pub closed enum Migration derives Copy, Eq, Encode {
    Identical
    Additive
    Migrated
} Migration

pub closed struct Widening derives Encode {
    pub holder: Principal
    pub step: u32
    pub authority: text::String
} Widening

pub closed struct Delivery derives Encode {
    pub step: u32
    pub kind: DeliveryKind
    /// The principal, or the instance as `inst.<moniker>`, whose authority performs it.
    pub issuer: Principal
    /// `<moniker>.<operation>`, or `<capability kind>.<operation>`.
    pub target: text::String
    /// The code hash of the bundle that handles a `Message` delivery, its target's, or runs a `Spawn` delivery, its issuer's; `None` for a `Capability` delivery.
    pub code: Option[HashValue]
    /// The host and port of an `Egress` delivery; else empty.
    pub destination: text::String
    pub size_bytes: u64
    pub blake3: HashValue
    pub class: Reversibility
} Delivery

pub closed enum DeliveryKind derives Copy, Eq, Encode {
    Message
    Capability
    Spawn
} DeliveryKind

pub closed struct Fault derives Encode {
    pub moniker: Moniker
    pub step: u32
    pub kind: FaultKind
    pub report: text::String
} Fault

pub closed enum FaultKind derives Copy, Eq, Encode {
    Trap
    Unhealthy
    Opaque
} FaultKind

pub closed struct ApprovalNeed derives Encode {
    /// The index of the deciding rule in `/sys/policy`.
    pub rule: u32
    pub decision: text::String
    pub reason: text::String
    pub approvers: mem::List[text::String]
} ApprovalNeed

pub closed struct TransportCut derives Encode {
    pub session: SessionId
    pub principal: Principal
    pub step: u32
} TransportCut

pub closed struct Costs derives Encode {
    pub cpu_ns: u64
    pub memory_bytes: i64
    pub store_bytes: u64
    pub energy_mj: u64
} Costs
```

- `steps`: each step's span, class and basis: `Observed` when a forked run or a tree step produced the class, `Declared` when it comes from a declaration; `reply` renders as a value of the operation's reply type.
- `delta`: every record and blob the plan writes or removes, in path order; a change prints `before` and `after` as `-` and `+` lines of the fields that differ, or of the blob ids.
- `activations`: each [instance](06-execution.md#instances) a step activates, its kind being the reconciliation effect of the change ([instance specifications](06-execution.md#instance-specifications)), `Rebuild` for an Env rebuild, `Restart` for a `restore-checkpoint` step and `Reboot` for a `generation` step, with the migration class of an update ([admission](10-code.md#admission)) and the expected pause.
- `widenings`: authority a holder gains: a [grant](09-authority.md#grants) beyond what the holder held at the base, an admission whose authority summary exceeds the previous [bundle](10-code.md#artefacts-and-bundles)'s, or a revert restoring a revoked grant.
- `deliveries`: held outputs and held spawns of class above `none`, and requests to instances holding resources. Those of irreversible class are the plan's outbox entries.
- `faults`: traps, `Unhealthy` replies and `opaque` results of forked runs, the report giving the trap kind and site or the reason.
- `approvals`: the policy's `ask`, `notify` and `deny` decisions with the deciding rule, among them the rule of two ([rule of two](19-oversight.md#rule-of-two)).
- `cuts`: each open [session](18-link.md#connections-and-sessions) whose transport a step can cut ([provisional commits](#provisional-commits)).
- `costs`: the CPU time and energy of applying, estimated from the forked runs, the change of memory charged, and the store bytes written, history included.

The link prints an effect set as records in field order: a `plan` summary, then `step`, `change`, `activation`, `widening`, `delivery`, `fault`, `approval`, `cut` and `cost` records, each element's first field being its record's subject and `cost` having none. A plan whose step 2 targets an instance step 1 reconfigures has two spans:

```text
> 6 plan [write /sys/config/app.feeds.batch 32, call app.feeds refresh()]
ok 6 plan [2 steps]: records 7
plan age7pwuq3zqxfkwosw, base: #5130, branch: "main", spans: 2, class: restorable, approvals: 0
step 1, span: 1, class: restorable, basis: Basis.Observed
step 2, span: 2, class: recorded, basis: Basis.Observed, reply: Refreshed(items: 412)
change /sys/config/app.feeds, step: 1
    - batch: 0
    + batch: 32
change /data/inst.app.feeds/digest, step: 2
    - count: 0
    + count: 412
activation app.feeds, step: 1, kind: ActivationKind.Rebuild, pause_ms: 2
cost cpu_ns: 3000000, memory_bytes: 0, store_bytes: 18432, energy_mj: 41
ok 6 plan [2 steps]: records 7
```

## The plan hash

A previewed plan is a `Plan` record:

```nan
pub closed struct Plan derives Encode {
    pub principal: Principal
    /// The canonical text of each step.
    pub steps: mem::List[text::String]
    pub options: PlanOptions
    pub effects: EffectSet
    pub inputs: RecordedInputs
} Plan

pub closed struct PlanOptions derives Encode {
    pub intent: Option[text::String]
    pub confirm_within_s: Option[u32]
    pub health_s: u32
} PlanOptions

pub closed struct RecordedInputs derives Encode {
    pub reads: mem::List[NodeRead]
    pub live: mem::List[LiveRead]
    pub values: mem::List[RecordedValue]
} RecordedInputs

pub closed struct NodeRead derives Encode {
    pub path: Path
    /// The commit that last wrote the node; `#0` when it is absent.
    pub commit: CommitId
} NodeRead

pub closed struct LiveRead derives Encode {
    pub path: Path
    pub hash: HashValue
} LiveRead

pub closed struct RecordedValue derives Encode {
    pub step: u32
    pub source: ValueSource
    pub value: mem::Bytes
} RecordedValue

pub closed enum ValueSource derives Copy, Eq, Encode {
    Store
    Clock
    Random
    Capability
} ValueSource
```

The **plan hash** is the hash of kind `plan` ([hashes](02-identifiers.md#hashes)) of the `mem::wire` encoding of the `Plan` record with every field of `effects.costs` zero. A plan is named by its hash's short form. The **read-set hash**, which approval decisions hold ([approvals](19-oversight.md#approvals)), is the hash of kind `read-set` of the encoding of `inputs.reads`.

## Apply

`apply <plan>` applies a previewed plan by its hash, under the authority of the [principal](09-authority.md#principals) that previewed it; another principal's `apply` fails with `plan.foreign`, and an `apply` of a plan applying under another request key with `plan.busy`. A plan applies at most once: an `apply` of a plan whose lifecycle has ended returns its final records under the request key it applied with, and fails with `plan.ended` under another.

### Spans

A **point of no return** is an irreversible step of a plan. A **span** is a maximal run of steps of class at most `restorable` followed by at most one point of no return; a span also ends before a `call` or `redeliver` step whose target an earlier step of the span activates. Steps keep their order. Each span is one [commit](11-store.md#transactions).

### Procedure

Apply runs these steps for each span in order:

1. Authorize: the principal's [grants](09-authority.md#grants) still cover every step, and `policy` allows the plan with the approvals present. A missing approval moves the plan to `awaiting-approval` ([plan lifecycle](#plan-lifecycle)); a `Deny` decision fails with `policy.denied` ([evaluation](19-oversight.md#evaluation)), and a lost grant with its authority code.
2. Validate: each node of the read set is at its recorded commit or at the commit of an earlier span of this plan, and each `/live` read hashes as recorded; otherwise the plan fails with `plan.stale`.
3. Pause the intakes of the span's targets, the [instances](06-execution.md#instances) it activates and those its `call` and `redeliver` steps name, in [moniker](02-identifiers.md#monikers) order, waiting for running handlers to return, within 1 s in total (`PlanConfig.activation_ms`); otherwise the paused intakes resume and the plan fails with `plan.activation-timeout`.
4. Replay each `call` and `redeliver` step of the span on its real target, its nondeterministic inputs served from the recording. Its store writes, held outputs, held spawns and reply must equal the preview's byte for byte; otherwise the intakes resume and the plan fails with `plan.diverged`. Steps targeting instances that hold resources are not replayed.
5. Commit, on the plan's [branch](11-store.md#branches), the span's tree steps, the replayed store writes, one `Held` outbox entry per irreversible delivery and, for a provisional span, the plan's `Plan` record at `/sys/plans/<hash>`, with the plan hash, the intent and the request key `<key>.<span>` ([transactions](11-store.md#transactions)). The committed delta is exactly the span's part of the previewed delta. A commit that fails ends the plan with the store's code; when it failed with `store.io` and the next mount replays it ([commit procedure](11-store.md#commit-procedure)), `link` treats it as a span commit of an interrupted plan.
6. Resume the intakes. `supervisor` activates the instances the span changes, and the span's held outputs and held spawns of class at most `restorable`, and its requests to instances holding resources, are released.
7. Run the span's health window, then wait for its confirmation when it is provisional.
8. Set the span's outbox entries `Delivering` in one commit, then deliver them in order ([the outbox](#the-outbox)); the next span starts once they are delivered.

A failure in steps 1 to 4 commits nothing of the span; earlier spans stay committed. When `link` restarts or the machine reboots during apply, the plan fails with `plan.interrupted`. When `link` starts again, it sets `Cancelled` the outbox entries of each committed span of the plan whose entries are all still `Held`, reverts each span commit of the plan that awaits its confirmation ([provisional commits](#provisional-commits)) as revert by blame does, and still delivers the plan's `Delivering` entries. A plan with a `generation` step is `applied` when its activation starts the reboot; the [health gate](03-boot.md#health-gate) decides the new generation.

Effects that held outputs cause after their release are commits and deliveries of their receivers, carrying the plan's [cause](17-observability.md#causes), outside the previewed delta and bounded by the receiving operations' declared classes.

### The outbox

The **outbox** `/sys/outbox/<id>` holds the irreversible deliveries of committed spans, `<id>` being `<plan short hash>-<span>-<n>`:

```nan
pub closed struct OutboxEntry derives Encode {
    pub plan: HashValue
    pub span: u32
    pub delivery: Delivery
    pub key: text::String
    /// The encoded message or capability arguments; empty for a spawn.
    pub payload: mem::Bytes
    pub status: DeliveryStatus
    pub attempts: u32
    pub reply: Option[mem::Bytes]
} OutboxEntry

pub closed enum DeliveryStatus derives Copy, Eq, Encode {
    Held
    Delivering
    Delivered
    Failed
    Cancelled
} DeliveryStatus
```

| State | Event | Next | Effect |
| --- | --- | --- | --- |
| — | its span commits ([procedure](#procedure) step 5) | `Held` | — |
| `Held` | [procedure](#procedure) step 8 reaches its span | `Delivering` | the delivery is sent once the span's earlier entries are `Delivered` |
| `Held` | its span's commit is reverted ([health windows](#health-windows), [provisional commits](#provisional-commits)), or `link` starts again ([procedure](#procedure)) | `Cancelled` | — |
| `Delivering` | its issuer or `link` restarts, for a `Message` or `Capability` | `Delivering` | it is delivered again; the receiving runner drops a delivery whose key it has handled since its start |
| `Delivering` | its target is restarting | `Delivering` | it is retried after the [restart](06-execution.md#restarts) backoff, at most 5 attempts in all |
| `Delivering` | it completes | `Delivered` | `reply` holds its reply, when it has one; `plan.delivered` |
| `Delivering` | it fails, its attempts run out, its issuer no longer holds the capability, or, for a `Spawn`, its issuer or `link` restarts | `Failed` | `plan.delivered`; the span's entries after it become `Cancelled`, and the plan fails with `plan.delivery-failed` ([plan lifecycle](#plan-lifecycle)) and keeps its commits |

- A `Message` delivery sends the payload to the target operation through the issuer's port, or, when the target is the issuer itself, appends it to the issuer's intake once the delay of its `Cx.after`, if any, has passed; a `Capability` delivery performs the operation on the issuer's capability; a `Spawn` delivery starts the held spawn in the issuer. After a restart the issuer's Env is rebuilt from its grants.
- A `Message` delivery fails when its target's bundle is not its `code`, and a `Spawn` delivery when its issuer's is not; a `Message` delivery still in its target's intake when the target changes bundle fails instead of passing to the new bundle ([transfer](07-communication.md#transfer)).
- The audit entry of an irreversible step is durable before any of its deliveries ([audit chain](17-observability.md#audit-chain)).
- A delivery is released only when the BLAKE3 of its payload equals the previewed one.
- Delivery is at least once.
- The key of the n-th delivery of span s is `<key>.<s>.<n>`.

## Plan lifecycle

| State | Event | Next | Effect |
| --- | --- | --- | --- |
| — | a preview completes | `previewed` | — |
| `previewed` | `apply`, every approval present | `applying` | — |
| `previewed` | `apply` or `call`, an approval missing | `awaiting-approval` | `policy` writes the approval ([approvals](19-oversight.md#approvals)) and `link` writes `/sys/plans/<hash>`; a `call` fails with `policy.approval-required` |
| `previewed` | `apply --when`, every approval present | `scheduled` | one commit writes `/sys/schedule/<id>` and `/sys/plans/<hash>` |
| `previewed`, `applying` | the policy decides `Deny` | `denied` | failure `policy.denied` |
| `previewed`, `applying` | a step is outside the principal's grants | `failed` | its authority code |
| `awaiting-approval` | the approval allows while an `apply` waits | `applying`, or `scheduled` with `--when` before its time | — |
| `awaiting-approval` | `apply` after the approval allows | `applying`, or `scheduled` with `--when` before its time | — |
| `awaiting-approval` | the approval denies | `denied` | failure `policy.approval-denied` |
| `awaiting-approval` | the approval expires | `expired` | failure `policy.approval-expired` |
| `awaiting-approval`, `scheduled` | a commit changes a node of the read set | `failed` | `plan.stale`; a pending approval lapses |
| `awaiting-approval`, `scheduled` | `cancel <plan>`, or `freeze agents` for a plan of a principal of an agent's [family](19-oversight.md#enrollment) | `failed` | `plan.cancelled` |
| `scheduled` | its time arrives | `applying` | — |
| `scheduled` | the time passed beyond the grace while the machine was off | `expired` | `plan.late` |
| `applying` | a span commits | `health-window` | activations |
| `applying` | a span's authorization finds an approval missing | `awaiting-approval` | `policy` writes the approval and `link` writes `/sys/plans/<hash>` |
| `health-window` | the window passes and the span is confirmed | `applying` | the span's deliveries |
| `applying` | the last span's deliveries end | `applied` | — |
| `applying` | validation, pause, replay or the span's commit fails | `failed` | `plan.stale`, `plan.activation-timeout`, `plan.diverged` or the store's code |
| `applying` | a delivery fails | `failed` | `plan.delivery-failed` |
| `health-window` | an activated [instance](06-execution.md#instances) fails | `reverted` | revert by blame; `plan.unhealthy` |
| `health-window` | the confirmation deadline passes | `reverted` | the revert; `plan.unconfirmed` |
| `health-window` | the span's revert conflicts with the commits after it | `failed` | `conflict.revert` |
| `applying` | `link` restarts or the machine reboots | `failed` | `plan.interrupted` |
| `health-window` | `link` restarts or the machine reboots | `failed` | `plan.interrupted`; when `link` starts again, the span's `Held` outbox entries become `Cancelled`, and a span commit awaiting its confirmation is reverted |

Each transition emits the event `plan.<state>` of its next state. `apply` answers `pending` until the plan ends, then gives its final response ([responses](18-link.md#responses)). `/live/plans/<hash>` holds a `LivePlan` for each plan the link holds, until 1 h after it ends. `/sys/plans/<hash>` holds the `Plan` record of a plan while it is `awaiting-approval` or `scheduled`, or while a span commit of it awaits its confirmation; the commit that ends that state deletes it.

```nan
pub closed struct PlanEvent derives Encode {
    pub plan: HashValue
    pub state: PlanState
    pub span: u32
    pub commit: Option[CommitId]
    /// The outbox entry of a `plan.delivered` event.
    pub entry: Option[text::String]
    pub code: Option[text::String]
} PlanEvent

pub closed struct LivePlan derives Encode {
    pub state: PlanState
    pub span: u32
    pub commits: mem::List[CommitId]
    /// The end of the running health window or confirmation wait.
    pub until: Option[WallTime]
    /// The plan's outbox entries.
    pub deliveries: mem::List[text::String]
    pub as_of: u64
} LivePlan

pub closed enum PlanState derives Copy, Eq, Encode {
    Previewed
    AwaitingApproval
    Scheduled
    Applying
    HealthWindow
    Applied
    Reverted
    Denied
    Expired
    Failed
} PlanState
```

## Health windows

A span's **health window** starts when its activations complete and lasts `--health` seconds. `supervisor` sends `Health` to each activated [instance](06-execution.md#instances) at the start and every 10 s, and counts its traps. A trap, an `Unhealthy` reply, no reply within 1 s, the instance entering `failed`, or the abort of its [live update](10-code.md#live-update) is a failure; an abort counts at the window's start. A span without activations has an empty window.

**Revert by blame**: a failure reverts the [commit](11-store.md#transactions) of the most recent span whose open window covers the failing instance. `link` commits the revert, `supervisor` reconciles, the span's outbox entries become `Cancelled`, and that span's plan alone ends `reverted` with `plan.unhealthy`. A revert that conflicts with the commits after it fails with `conflict.revert` ([revert](11-store.md#revert-and-diff)) and ends the plan `failed`.

## Provisional commits

With `--confirm-within <s>`, each span's [commit](11-store.md#transactions) is **provisional**: unless `confirm <commit>` arrives from the plan's [principal](09-authority.md#principals), over any of its [sessions](18-link.md#connections-and-sessions), within s seconds of the end of the span's activations, `link` reverts it as revert by blame does and the plan ends `reverted` with `plan.unconfirmed`. A provisional span commit awaits its confirmation while the `Plan` record it wrote ([procedure](#procedure)) is at `/sys/plans/<hash>`; its confirmation commits the deletion of that record. The deliveries and the next span of a provisional span wait for its confirmation. A `confirm` of a commit that is not provisional or belongs to another principal fails with `plan.not-provisional`.

A step can **cut** a session when it activates, other than by `Start`, an [instance](06-execution.md#instances) on the session's path ([connections and sessions](18-link.md#connections-and-sessions)); a write of `/sys/config/net` cuts exactly the sessions that [provisional changes](16-network.md#provisional-changes) names. The effect set lists each open session a step can cut.

## Undo

`undo <commit or plan>` previews the inverse plan and commits nothing; `apply` applies it like any plan.

- The inverse of a commit is `revert <commit>`, with the activations that reconcile the reverted state.
- The inverse of a plan reverts its span commits and every commit carrying its [cause](17-observability.md#causes) other than those writing `/sys/boot`, newest first, then calls, for each delivered outbox entry newest first, its compensation when the target's protocol declares one.
- While the generation a plan's `generation` step installed runs, the inverse of the plan ends with `call boot rollback()` ([generation operations](03-boot.md#generation-operations)), which only `owner` invokes ([emergency controls](19-oversight.md#emergency-controls)).
- A delivered entry without a compensation prints as an `irreversible` record: no undo reaches it.
- The inverse of an `update` follows [rollback after probation](10-code.md#rollback-after-probation): lossless when the schemas allow it, else a `restore-checkpoint` step listing the commits whose state it drops.

## Request keys

A **request key** names a mutating request: `apply`, `call`, `confirm`, `cancel`, `put`, `extract`, `delegate`, `approve` and `freeze`. It is the request's `--key <key>`, of 1 to 64 characters of `[A-Za-z0-9_.-]`, by default the session in its `s<b>.<n>` form, `.` and the request id (`s12.3.7`).

- `link` keeps the result of the last 1,024 mutating requests of each [session](18-link.md#connections-and-sessions) (`PlanConfig.key_results`), with each request's identity: its verb, operands and flags other than `--session`, `--budget`, `--skip`, `--json` and `--fields`. The kept result of a `delegate` omits the `seed` of its `credential` record, so a repeated `delegate` prints that record without it, also while the earlier one is pending.
- A request whose key a kept result of the [principal](09-authority.md#principals)'s sessions holds gets that result when its identity is equal; when the earlier request is still pending, it gets that request's state, and the final response follows. A different identity fails with `request.key-reused` ([codes](18-link.md#codes)).
- The key of a span [commit](11-store.md#transactions) is `<key>.<span>`. An `apply` whose key a span commit carries returns the plan's state and commits nothing.

## Concurrency

Commits are optimistic: each span validates the plan's read set at its [commit](11-store.md#transactions), and other writers commit freely between spans. Agents coordinate through branches, merges and `--expect` ([branches](11-store.md#branches)); no lock exists. Activation pauses intakes in [moniker](02-identifiers.md#monikers) order for at most `PlanConfig.activation_ms`, and two plans pausing a common [instance](06-execution.md#instances) pause it in that order.

## Timed plans

`apply <plan> --when <time>`, the time being RFC 3339 UTC or `+<n>s` from now, schedules the plan; approvals are obtained before it is `scheduled`.

```nan
pub closed struct TimedPlan derives Encode {
    pub plan: HashValue
    pub principal: Principal
    pub at: WallTime
    pub key: text::String
} TimedPlan
```

- `/sys/schedule/<id>` holds a `TimedPlan`, `<id>` being the plan's short hash.
- When the wall clock less its uncertainty reaches `at` ([clocks](08-scheduling.md#clocks)), `link` applies the plan as previewed under its [principal](09-authority.md#principals)'s [grants](09-authority.md#grants) at that moment, with the key `key`.
- A time that passed while the machine was off applies at the next boot when it passed at most 1 h before (`PlanConfig.grace_s`); otherwise the plan ends `expired` with `plan.late`.
- `cancel <plan>` deletes the entry, and so does reverting the [commit](11-store.md#transactions) that wrote it.

## Events

| Kind | Payload | Section |
| --- | --- | --- |
| `plan.previewed`, `plan.awaiting-approval`, `plan.scheduled`, `plan.applying`, `plan.health-window`, `plan.applied`, `plan.reverted`, `plan.denied`, `plan.expired`, `plan.failed` | `PlanEvent` | [Plan lifecycle](#plan-lifecycle) |
| `plan.delivered` | `PlanEvent` | [The outbox](#the-outbox) |

## Codes

| Code | Meaning | Section |
| --- | --- | --- |
| `plan.activation-timeout` | the intakes of a span's targets did not pause in time | [Procedure](#procedure) |
| `plan.busy` | the plan is applying under another request key | [Apply](#apply) |
| `plan.ended` | the plan's lifecycle ended under another request key | [Apply](#apply) |
| `plan.cancelled` | the plan was cancelled while awaiting approval or scheduled | [Plan lifecycle](#plan-lifecycle) |
| `plan.delivery-failed` | an outbox delivery failed; the plan's commits stay | [The outbox](#the-outbox) |
| `plan.diverged` | a replayed step's outputs differ from the preview's | [Procedure](#procedure) |
| `plan.foreign` | the plan was previewed by another principal | [Apply](#apply) |
| `plan.horizon` | a step depends on the real result of an earlier irreversible step | [Horizon](#horizon) |
| `plan.interrupted` | `link` restarted or the machine rebooted during apply | [Procedure](#procedure) |
| `plan.late` | a timed plan's time passed beyond the grace while the machine was off | [Timed plans](#timed-plans) |
| `plan.not-provisional` | the commit is not a provisional commit of the principal | [Provisional commits](#provisional-commits) |
| `plan.preview-too-large` | the plan's forked copies and recorded inputs exceed the bound | [Forked runs](#forked-runs) |
| `plan.stale` | a node of the read set, a `/live` read or an `--expect` changed since the preview | [Procedure](#procedure), [Steps](#steps) |
| `plan.step-misplaced` | a `generation` step is not last, or a step needs the running branch | [Steps](#steps) |
| `plan.unconfirmed` | a provisional commit was not confirmed in time | [Provisional commits](#provisional-commits) |
| `plan.unhealthy` | an activated instance failed in its health window | [Health windows](#health-windows) |
