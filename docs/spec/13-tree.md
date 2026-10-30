# The tree

This chapter defines the tree: its top-level nodes, the nodes below `/sys`, `/data`, `/state`, `/journal` and `/history` with their kinds, types and writers, the system definition, desired and observed state with the reconciliation status, `/live` and its nodes, path resolution relative to capabilities, and genesis.

## Top-level nodes

The **tree** is the namespace of NaN OS: the store's nodes ([11-store.md](11-store.md#nodes)) and `/live`. Paths are those of [02-identifiers.md](02-identifiers.md#paths).

| Node | Holds | Kept |
| --- | --- | --- |
| `/sys` | the system definition | on the running branch, versioned |
| `/data` | principals' data | on the running branch, versioned |
| `/state` | what the runner writes for each instance | on the running branch, [unversioned](11-store.md#unversioned-nodes) |
| `/journal` | the journal's streams | in the machine tree, unversioned |
| `/history` | commits, branches and pins | in the store's commit metadata, read-only |
| `/live` | observed state | nowhere: its providers publish it, read-only |

Another top-level name fails with `store.not-found`, and a write below `/history` or `/live` with `store.denied` ([11-store.md](11-store.md#codes)). The dirs of `/sys` declare the element types of their children ([11-store.md](11-store.md#declared-types-and-validity)), so a record of another type never lands there.

## /sys

Every write below `/sys` is a serializable commit recorded in history ([11-store.md](11-store.md#isolation)). [Plans](14-changes.md#apply) write it under the [policy](19-oversight.md#policy); the table names its other writers.

| Path | Kind and type | Written by | Read by |
| --- | --- | --- | --- |
| `/sys/instances/<moniker>` | record: an instance's [specification](06-execution.md#instance-specifications) | plans; the kernel at genesis and generation boots; `devices` for drivers ([driver instances](15-devices.md#driver-instances)); `supervisor` for update reverts ([probation](10-code.md#probation)) | `supervisor` |
| `/sys/config/<moniker>` | record of the [configuration type](09-authority.md#env-construction) of the instance's component, or at `/sys/config/auth`, `auth` naming no [instance](06-execution.md#instances), the `AuthConfig` of [delegation](09-authority.md#delegation) ([/sys/config](#sysconfig)) | plans; the kernel at genesis and generation boots; `policy` for the machine id ([owner](19-oversight.md#owner)); `supervisor` for `SupervisorConfig.frozen` ([emergency controls](19-oversight.md#emergency-controls)) | the runner, into the instance's Env ([Env construction](09-authority.md#env-construction)); `supervisor`, which activates the instance on a write ([14-changes.md](14-changes.md#apply)); `link` for `/sys/config/auth` |
| `/sys/code/<hash>` | dir named by a bundle's code hash, holding `admission`, `bundle` and `image` ([10-code.md](10-code.md#admission-record)) | `admission` | `admission`, `supervisor`, the store's schema registry |
| `/sys/principals/<principal>` | record: a principal record ([09-authority.md](09-authority.md#principals)) | `keys`; `link` for `delegate` ([delegation](09-authority.md#delegation)); genesis | `link`, `keys`, `policy` |
| `/sys/grants/<id>` | record: a [grant](09-authority.md#grants) | plans; `link` for `delegate`; `keys` at claim; `devices` for drivers; `supervisor` for expiries ([revocation](09-authority.md#revocation)); genesis | the runner, `link`, `policy` |
| `/sys/policy` | record: the owner-signed [policy](19-oversight.md#policy) | plans; genesis | `policy` |
| `/sys/approvals/<id>` | record: an [approval](19-oversight.md#approvals) | `policy` | `link`, `console`, `policy` |
| `/sys/plans/<hash>` | record: a plan awaiting approval, scheduled, or with a span commit awaiting its confirmation ([14-changes.md](14-changes.md#plan-lifecycle)) | `link` | `link` |
| `/sys/outbox/<id>` | record: an [outbox](14-changes.md#the-outbox) entry | plans; `link` as it delivers | `link` |
| `/sys/schedule/<id>` | record: a [timed plan](14-changes.md#timed-plans) | `link` | `link` |
| `/sys/boot` | record `BootState` | `boot` | `boot`, `link` |

### /sys/config

These configuration records have the types their owning chapters define:

| Path | Record | Owner |
| --- | --- | --- |
| `/sys/config/auth` | `AuthConfig` | [09-authority.md](09-authority.md#delegation) |
| `/sys/config/boot` | `BootConfig` | [03-boot.md](03-boot.md#health-gate) |
| `/sys/config/clock` | `ClockConfig` | [20-base.md](20-base.md#clock) |
| `/sys/config/console` | `ConsoleConfig` | [20-base.md](20-base.md#console) |
| `/sys/config/devices` | `DevicesConfig`, the desired bindings | [15-devices.md](15-devices.md#the-device-manager) |
| `/sys/config/infer` | `InferConfig` | [20-base.md](20-base.md#infer) |
| `/sys/config/journal` | `JournalConfig` | [17-observability.md](17-observability.md#the-journal-and-its-streams) |
| `/sys/config/link` | `LinkConfig` | [18-link.md](18-link.md#configuration) |
| `/sys/config/net` | `NetConfig`, the desired network | [16-network.md](16-network.md#configuration) |
| `/sys/config/policy` | `PolicyConfig`, with the machine id | [19-oversight.md](19-oversight.md#owner) |
| `/sys/config/power` | `PowerConfig` | [08-scheduling.md](08-scheduling.md#frequency) |
| `/sys/config/store` | `StoreConfig { retention: Retention }` | [11-store.md](11-store.md#retention) |
| `/sys/config/supervisor` | `SupervisorConfig` | [20-base.md](20-base.md#supervisor) |

### /sys/boot

`BootState` mirrors what `boot` last wrote to the boot record and the boot volume ([03-boot.md](03-boot.md#boot-record)):

| Field | Type | Meaning |
| --- | --- | --- |
| `slots` | `mem::List[Slot]` | slots A and B, `Slot { name: text::String, generation: u64, state, tries_left: u8 }`, `name` being `a` or `b`, state `empty`, `trial`, `good` or `bad` |
| `recovery` | `u64` | the recovery generation |
| `floor` | `u64` | the anti-rollback floor `boot` last read |
| `floor_request` | `u64` | the floor raise requested of the loader, 0 for none |
| `generations` | `mem::List[Generation]` | the generations on the boot volume |

`Generation` is `{ number: u64, manifest: HashValue, kernel: HashValue, genesis: HashValue, min_loader: u32, store_format: u32, signer: Principal, installed: WallTime, base: CommitId }`: the manifest's fields ([03-boot.md](03-boot.md#generations-and-manifests)), the installation time, and `base`, the commit `boot` pins for it ([11-store.md](11-store.md#pins)).

## The system definition

The **system definition** is the content of `/sys` on the running branch. `/sys/instances` and `/sys/config` hold the desired state: what runs, and how. `/sys/code` holds the code admitted to run; `/sys/principals`, `/sys/grants` and `/sys/policy` say who may act; `/sys/approvals`, `/sys/plans`, `/sys/outbox` and `/sys/schedule` hold the state of plans; `/sys/boot` mirrors the boot volume. Nothing else configures the machine, except the boot volume's files ([03-boot.md](03-boot.md#boot-volume)) and the constants of the loader and the kernel image. Every change to the system definition is a commit carrying its principal, cause and plan, so `history /sys` lists every retained configuration the machine ran, and `diff` and `revert` reach each of them.

## /data

`/data/<principal>` holds one principal's data ([02-identifiers.md](02-identifiers.md#principals)): any nodes, declared as that principal chooses. `keys` creates it at [enrollment](19-oversight.md#enrollment), and the instance principal `inst.<moniker>` has one when a grant names it. `/data/<principal>/inbox/<child>` is where a delegated child principal writes its results: `delegate` ([18-link.md](18-link.md#verbs)) creates it and grants the child `write` on it, and its nodes carry the child's origin set.

## /state

`/state/<moniker>` holds what the runner writes for one instance, as the instance's principal. Its nodes are unversioned, and [06-execution.md](06-execution.md#checkpoints) sets how many of each the runner keeps:

| Path | Kind |
| --- | --- |
| `checkpoint/<n>` | records of a numbered dir with `keep`: the [checkpoints](06-execution.md#checkpoints), the pre-update one held beyond that number while its [probation](10-code.md#probation) runs |
| `dead/<seq>` | records of a numbered dir with `keep`: the [dead letters](06-execution.md#dead-letters) |
| `job`, `result` | records: a [job](06-execution.md#jobs)'s specification and result |
| `stdin` | blob: a job's input |
| `stdout`, `stderr` | streams of a job's output |
| `labels` | record: the instance's ingress labels ([19-oversight.md](19-oversight.md#origin-sets)) |
| `recording` | blob: the instance's last flight recording ([17-observability.md](17-observability.md#flight-recorders)) |

`restore-checkpoint` and `redeliver` ([14-changes.md](14-changes.md#steps)) name a child of `checkpoint` or `dead` by its number. `/state/keys` holds the state of `keys`, which no grant reaches but that of `keys` itself ([09-authority.md](09-authority.md#grants)).

## /journal

`/journal` holds six streams in the machine tree, written only by `journal`: `events`, `audit`, `log`, `metrics`, `traces` and `faults`. Their entry types and retention are in [17-observability.md](17-observability.md#the-journal-and-its-streams). Every branch sees the same streams.

## /history

`/history` renders the store's commit metadata ([11-store.md](11-store.md#commits)):

| Path | Holds |
| --- | --- |
| `/history/commits/<n>` | the `CommitMeta` of retained commit n |
| `/history/branches/<name>` | the `Branch` record of a branch, `<name>` taking as many segments as the name has |
| `/history/pins/<n>` | the pins on commit n |

`/history/commits` and `/history/pins` list in numeric order. A commit appears to a View only when it changed a node inside the View's subtree, and shows only those paths.

## Desired and observed state

Three instances reconcile observed state with desired state: `supervisor` makes `/live/instances` match `/sys/instances` and `/sys/config` ([06-execution.md](06-execution.md#lifecycle)); `devices` makes `/live/devices` match `/sys/config/devices` ([15-devices.md](15-devices.md#the-device-manager)); `net` makes `/live/net` match `/sys/config/net` ([16-network.md](16-network.md#configuration)). Each subscribes to its desired nodes ([11-store.md](11-store.md#the-store-protocol)) and gives every node it reconciles a status, `ReconcileStatus { state, desired: u64, since: WallTime }`: `state` is `converged`, `converging`, or `failed` with a code and a detail text; `desired` is the commit of the desired value it acts on; `since` is the time of the last change.

| State | Event | Next | Effect |
| --- | --- | --- | --- |
| `converged` | a commit changes the desired value | `converging` | `desired` takes the commit; the reconciler acts |
| `converging` | the observed state matches the desired value | `converged` | — |
| `converging` | a commit changes the desired value | `converging` | `desired` takes the newer commit |
| `converging` | an action fails and the reconciler gives up | `failed` | the reconciler emits its failure event |
| `failed` | a commit changes the desired value | `converging` | the reconciler acts again |

An imperative operation of these three instances, such as `call supervisor stop(...)`, which sets `InstanceSpec.stopped`, `call devices bind(...)` or `call net configure(...)`, is a plan step committing the desired value, and it ends when the node's status reports `converged` or `failed` for that commit. Each reconciler states when it gives up an action.

## /live

`/live` holds observed state and is never stored. Each node holds a record that its provider publishes, and may have child nodes. An instance publishes its summary with `Cx.publish` ([06-execution.md](06-execution.md#held-effects)); the runner and the core read their counters at request time. A read returns the last published record with `as_of`, the monotonic time of its publication ([02-identifiers.md](02-identifiers.md#time)); it never waits for, revives or pauses an instance, and costs no storage and no wake-up. Records of one response are not read atomically together. A `/live` path resolves only through a View granted on it ([09-authority.md](09-authority.md#capability-kinds)), with `read` rights at most.

| Path | Record | Provider |
| --- | --- | --- |
| `/live/status` | `LiveStatus` | the core |
| `/live/machine` | `Machine` ([01-machines.md](01-machines.md#measured-facts)) | the core |
| `/live/instances/<moniker>` | `LiveInstance` | the runner and `supervisor` |
| `/live/sessions/<session>` | `LiveSession` ([18-link.md](18-link.md#connections-and-sessions)) | `link` |
| `/live/plans/<hash>` | `LivePlan` ([14-changes.md](14-changes.md#plan-lifecycle)) | `link` |
| `/live/devices/<bus path>` | `LiveDevice` ([15-devices.md](15-devices.md#the-device-manager)), with the units and partitions of [volumes and roles](15-devices.md#volumes-and-roles) below it | `devices` |
| `/live/cpus/<n>` | `LiveCpu` | the core |
| `/live/memory` | `MemorySummary` ([05-memory.md](05-memory.md#memory-summary)) | the core |
| `/live/accounts/<account>` | `Account` | the core |
| `/live/net` | `LiveNet`, with `LiveFlow` records below it ([16-network.md](16-network.md#configuration)) | `net` |
| `/live/security` | `LiveSecurity` | the core, `keys`, `boot` and `journal` |
| `/live/boot` | `LiveBoot` | `boot` |
| `/live/store` | `LiveStore` | `store` |

Jobs are instances and appear under `/live/instances`. An [account](08-scheduling.md#accounting) is named by its instance's moniker or its session's id. The records this chapter defines follow; each ends with `as_of: u64`.

`LiveStatus`: `generation: u64`; `mode`, one of `normal`, `trial`, `recovery` and `dev` ([03-boot.md](03-boot.md#modes)); `claimed: bool`; `uptime: u64`; `time: WallTime`; `instances: { running: u32, converging: u32, failed: u32 }`; `approvals: u32`, those pending; `store_used_pct: u8`, the share of the store's capacity in use; `exposures: mem::List[text::String]`, the `LiveSecurity` fields away from their protected value; `cursor: Cursor`, the [event cursor](02-identifiers.md#identifiers) of the latest event.

`LiveInstance`:

| Field | Type | Meaning |
| --- | --- | --- |
| `state` | `Lifecycle` | its lifecycle state ([06-execution.md](06-execution.md#lifecycle)) |
| `status` | `ReconcileStatus` | |
| `bundle` | `HashValue` | the code hash of the bundle it runs |
| `space` | `text::String` | `shared`, `keys`, or its isolated space `<principal>/<class>` ([09-authority.md](09-authority.md#placement)) |
| `durability` | `Durability` | `volatile` or `checkpointed` ([06-execution.md](06-execution.md#durability)) |
| `restarts` | `u32` | restarts in the last 60 s |
| `fault` | `Option[text::String]` | the code of its last trap or fault |
| `intake` | `u32` | queued messages |
| `memory` | `Usage { charged: u64, limit: u64, peak: u64 }` | its cell |
| `cpu_ns` | `u64` | CPU time charged to its account |
| `probation` | `Option[WallTime]` | the end of a [probation](10-code.md#probation) in progress |
| `summary` | `Option[TypedValue]` | the summary it last published |

`LiveCpu`: `online: bool`; `package`, `core`, `llc` and `node`, each `u32`, its place in the topology the core records ([04-kernel.md](04-kernel.md#cpus)); `siblings: mem::List[u32]`; `time: { normal: u64, background: u64, latency: u64, interrupts: u64, idle: u64 }`, by [CPU class](08-scheduling.md#cpu-classes), for interrupt and core work, and idle; `run_queue: u32`; `idle_state: text::String`; `frequency_mhz: u32`; `wakeups: u64`.

`Account`: `cpu_ns: u64`; `wakeups: u64`; `io_bytes: u64`; `energy_mj: u64`; `throttled: bool`, its energy budget spent ([08-scheduling.md](08-scheduling.md#energy)); `class`, its CPU class; `charge`, `peak` and `limit`, each `u64`, its cell ([05-memory.md](05-memory.md#cells-and-limits)).

`LiveSecurity`: `rollback`, `protected` or `unprotected` ([03-boot.md](03-boot.md#floor-and-owner-key-storage)); `loader`, `production` or `development` ([03-boot.md](03-boot.md#development-loader)); `secure_boot: bool`; `measured: bool`; `dma`, `confined` or `unconfined`; `isolated_spaces: bool`; `keys`, `isolated` or `unconfined`; `key_storage`, `tpm` or `software` ([09-authority.md](09-authority.md#keys-on-the-machine)); `link_keys: { x25519: mem::Bytes, ed25519: mem::Bytes }`, the public halves of the link key ([16-network.md](16-network.md#network-transports-of-the-link)); `audit`, `intact` or `broken`, and `audit_key: mem::Bytes`, the audit public key ([17-observability.md](17-observability.md#audit-chain)); `volume`, `encrypted` or `plain` ([12-store-format.md](12-store-format.md#volumes)). Each field's protected value is the one named before the other.

`LiveBoot` ([03-boot.md](03-boot.md#handoff-record)): `generation: u64`; `mode`; `slot: Option[text::String]`, the `name` of the booted slot, `a` or `b`, none when the machine booted the recovery generation or a development image; `boot_counter: u64`; `loader: u32`, the loader version; `floor: u64`; `loader_entry` and `kernel_entry: u64`, the time-stamp counter at the loader's entry, from the handoff record, and at the kernel's; `counter_hz: u64`, that counter's measured frequency; `support: HashValue`, the hash of the support-routine table the running kernel image exports ([bundle file](10-code.md#bundle-file)); `previous`, `healthy` or `unhealthy`; `failures: mem::List[text::String]`, the candidates the loader gave up, with their codes.

`LiveStore` ([12-store-format.md](12-store-format.md)): `placement: text::String`, a partition's bus path or `file`; `others: mem::List[text::String]`, the store partitions left unused; `capacity`, `used`, `history`, `agents` and `free`, each `u64`, `history` and `agents` counting the [history and agent bytes](11-store.md#thinning); `format: u32`; `encrypted: bool`; `head: u64`, the latest commit number; `branches: u32`; `pins: u32`; `replayed: u64`, the log bytes replayed at mount; `flushed: WallTime`, the latest index flush; `free_segments: u32`; `scrub: Option[ScrubReport]`, the latest one.

## Path resolution

1. A path names one node: below `/sys`, `/data` and `/state`, on a branch; below `/journal`, in the machine tree; below `/history` and `/live`, at their providers.
2. A request's path resolves through the capabilities of its session: its root capability and the Views minted from its grants ([09-authority.md](09-authority.md#sessions-as-authority)). The View whose subtree is the longest prefix of the path applies, with its branch, rights and as-of. The request's `--at` names another retained commit of that branch, or a branch the session holds a View on ([18-link.md](18-link.md#verbs)).
3. A path inside no View the session holds fails with `store.denied`, whether or not a node exists there.
4. NaN code reaches the tree only through capabilities in its Env: a `View` or a `sys::Fs` resolves paths relative to its dir, in NaN's relative form ([11-store.md](11-store.md#file-views)).
5. A name never confers authority. A moniker names an instance and is one path segment: `/sys/instances/app.mail` and `/state/app.mail`.

## Genesis

A generation's **genesis record**, the file `/nanos/gen/<n>/genesis` ([03-boot.md](03-boot.md#boot-volume)), is the system definition the kernel writes into an empty store, as wire bytes:

| Field | Type | Meaning |
| --- | --- | --- |
| `format` | `u32` | 1 |
| `generation` | `u64` | the manifest's generation |
| `nodes` | `mem::Map[Path, Declaration]` | the dirs and streams to create with their declarations: the dirs of `/sys`, `/data` and `/state`, and the streams of `/journal` |
| `instances` | `mem::Map[Moniker, InstanceSpec]` | the specifications of the [base instances](20-base.md#common-rules) |
| `admission` | `TypedValue` | the base bundle's [admission record](10-code.md#admission-record) |
| `config` | `mem::Map[Moniker, TypedValue]` | their configuration records, `store`'s holding the retention defaults, and under `auth` the `AuthConfig` of [delegation](09-authority.md#delegation) |
| `principals` | `mem::List[TypedValue]` | the vendor principals' records |
| `grants` | `mem::List[TypedValue]` | the root grant `g0` and the base instances' grants ([09-authority.md](09-authority.md#grants)) |
| `policy` | `TypedValue` | the preset `supervised` ([19-oversight.md](19-oversight.md#presets)) |

`TypedValue` is `{ type: TypeRef, bytes: mem::Bytes }`, a record value with its type ([11-store.md](11-store.md#declared-types-and-validity)).

1. When the store mounts a volume holding no commit, the kernel decodes the running generation's genesis record with its validator. A failure halts the kernel ([04-kernel.md](04-kernel.md#kernel-halts)).
2. The store creates `main` and commits #1, of kind `Genesis`, principal `system` and intent `genesis <n>`: the declared nodes, every specification, configuration record, principal record and grant, the policy, and `/sys/boot` from the [handoff record](03-boot.md#handoff-record). On a claimed machine, whose handoff record holds the owner key, it also writes the owner's principal record and grant and `/data/owner`; in `dev` mode on an unclaimed machine it writes them with no key, so the console's `hello owner` holds the owner's authority ([03-boot.md](03-boot.md#modes)). Genesis runs in every mode, recovery included. A failure halts the kernel.
3. `supervisor` reconciles `/sys/instances`.

At every boot after genesis, before `supervisor` reconciles, the kernel compares the base instances of the running generation's genesis record with the running branch. When they differ, it commits, as `system` with intent `generation <n>`, the specifications and configuration records of base monikers the branch lacks, the removal of specifications of base monikers the record no longer lists, and the kernel image's bundle in every base specification ([06-execution.md](06-execution.md#instance-specifications)); it touches no other node. A failure of that commit halts the kernel ([04-kernel.md](04-kernel.md#kernel-halts)). A claim completing after genesis writes the owner's records as [claim](19-oversight.md#claim) states.
