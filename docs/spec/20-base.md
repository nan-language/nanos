# Base instances

This chapter defines the base instances: the rules they share, their start order and lazy start, and for each one what it does, the operations of its protocol, its configuration and its wiring; then the jobs a generation ships and the requests that replace the classical utilities.

## Common rules

A **base instance** is an instance whose component is linked into the [kernel image](04-kernel.md#kernel-image).

- The base components form the kernel image's base bundle, of provenance `vendor`, and their specifications set `InstanceSpec.principal` to `system` ([base instances](10-code.md#base-instances)). Genesis writes their specifications, configuration records and grants ([genesis](13-tree.md#genesis)), and their top-level monikers are reserved ([instances](06-execution.md#instances)).
- They run in the shared space, except `keys`, which runs alone in an isolated space ([placement](09-authority.md#placement)).
- Their restart policy is `permanent` ([restarts](06-execution.md#restarts)).
- They update like any instance ([live update](10-code.md#live-update)) and all run in recovery mode ([recovery](03-boot.md#recovery)).
- A session reaches an operation as `call <moniker> <operation>(<args>)`, a one-step plan the policy decides ([evaluation](19-oversight.md#evaluation)); an operation the tables give a holder is masked off every other port ([ports](07-communication.md#ports)). A name naming nothing fails with `request.unknown-name` ([requests](18-link.md#requests)), which the tables omit.

| Moniker | Does | Durability | Start | Idle cost |
| --- | --- | --- | --- | --- |
| `supervisor` | reconciles instances | `volatile` | step 1 | none |
| `devices` | the device manager | `volatile` | step 2 | none |
| `fat` | the boot volume | `volatile` | step 3 | none |
| `keys` | keys and signatures | `checkpointed` | steps 4 and 6 | none |
| `store` | the store | `volatile` | step 5 | none |
| `journal` | the journal | `volatile` | step 6 | none |
| `policy` | decisions and approvals | `checkpointed` | step 6 | one timer per pending approval |
| `boot` | generations and the boot record | `volatile` | step 7 | none |
| `net` | the network stack and vsock, on a machine with `nic` or a vsock device | `checkpointed` | step 9 | the timers of [address configuration](16-network.md#address-configuration) |
| `link` | the link | `checkpointed` | step 9 | one timer per detached session |
| `console` | the human's console | `volatile` | step 9 | none |
| `admission` | admission and lowering | `volatile` | lazy | none |
| `clock` | wall time | `volatile` | lazy | the polls of its network time sources |
| `power` | power off, reboot, power settings | `volatile` | lazy | none |
| `infer` | model inference | `volatile` | lazy | none |

## Start order

1. The core's first task starts `supervisor` ([start sequence](04-kernel.md#start-sequence)).
2. `supervisor` starts `devices`, which binds the drivers of the `console`, `boot-volume`, `store-volume`, `entropy` and `tpm` roles before any other device ([volumes and roles](15-devices.md#volumes-and-roles)).
3. `fat` starts on the `boot-volume` role.
4. `keys` starts without reading its checkpoint and serves only the `volume` purpose ([keys on the machine](09-authority.md#keys-on-the-machine)).
5. `store` mounts its volume ([mounting and recovery](12-store-format.md#mounting-and-recovery)), committing genesis on an empty store ([genesis](13-tree.md#genesis)).
6. The runner restarts `keys` from its checkpoint; `journal`, then `policy`, start.
7. `boot` starts: it records the boot's outcome in the boot record and `/sys/boot`, and in `trial` mode runs the health gate ([health gate](03-boot.md#health-gate)).
8. `supervisor` reconciles `/sys/instances`, starting independent instances in parallel.
9. `net` starts on a machine with `nic` or a vsock device; `link` opens its transports, and `console` starts.

When `store`, `keys`, `journal` or `policy` is not `running` 30 s after the core's start, the kernel halts (`fault.base-not-running`), and the loader counts the boot ([health gate](03-boot.md#health-gate)). Another base instance that fails to start stays `failed` and shows in `/live/instances`.

The lazy base instances have `lazy` set, start at their first message ([lazy start](06-execution.md#lazy-start)) and count as `converged` before it.

## `supervisor`

`supervisor` reconciles `/sys/instances` and `/sys/config` with the instances that run ([Execution](06-execution.md)), supervises every top-level instance, and runs [health windows](14-changes.md#health-windows) and [probations](10-code.md#probation). Its operations write desired state ([desired and observed state](13-tree.md#desired-and-observed-state)):

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `start` | `moniker` | — | `instance.supervisor-missing`, `place.unavailable` | `restorable` |
| `stop` | `moniker` | — | — | `restorable` |
| `restart` | `moniker` | — | — | `restorable` |
| `hibernate` | `moniker` | — | `instance.volatile`, `instance.not-running` | `restorable` |
| `wake` | `moniker` | — | — | `restorable` |
| `redeliver` | `moniker`, `seq: u64` | — | `instance.no-dead-letter`, `instance.dead-letter-empty` | the class of the redelivered operation |
| `freeze` | `frozen: bool` | — | — | `restorable` |

- `start` and `stop` write `InstanceSpec.stopped`; they and `redeliver` are also plan steps ([steps](14-changes.md#steps)).
- `freeze` writes `SupervisorConfig { frozen: bool, probation: Probation }` at `/sys/config/supervisor`, `frozen` being `false` by default and `probation` holding the [probation](10-code.md#probation) defaults. While `frozen`, the intake of every instance other than a base instance stays paused. Only `console` holds `freeze` ([emergency controls](19-oversight.md#emergency-controls)).
- Wiring: a `Control` grant over every moniker; ports to `store`, `journal`, `admission` and `boot`.

## `devices`

The device manager: discovery, the driver instances `devices.*`, the AML evaluator `devices.aml` and the volume views ([Devices](15-devices.md)). Its operations `bind`, `unbind`, `power` and `reset` are those of [the device manager](15-devices.md#the-device-manager). The core drives the hardware [watchdog](04-kernel.md#watchdog). Wiring: the device root; a `Control` grant over `devices`; ports to `fat` and `journal`.

## `fat`

Access to the boot volume, with the operations of [the fat component](15-devices.md#the-fat-component). Wiring: the `boot-volume` role. `boot` and `devices` hold ports to it.

## `store`

The store ([The store](11-store.md)) on its volume ([Store format](12-store-format.md)), with the operations of [the Store protocol](11-store.md#the-store-protocol); the link's reading verbs read through it. Wiring: the `store-volume` role ([placement](12-store-format.md#placement)); the `KeyUse` purposes `volume`, which unseals an encrypted volume's key ([volumes](12-store-format.md#volumes)), and `node` ([keys on the machine](09-authority.md#keys-on-the-machine)); `sys::Random` for the seeds of secret records ([secret records](11-store.md#secret-records)); a port to `journal`.

## `keys`

Keys and signatures ([keys and signatures](09-authority.md#keys-and-signatures)). Besides the `KeyUse` purposes `handshake`, `anchor`, `node` and `volume` of [keys on the machine](09-authority.md#keys-on-the-machine), it serves:

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `verify` | `principal: Principal`, `kind: text`, `subject: HashValue`, `signature: Signature`, `former: bool` | — | `auth.bad-signature` | `none` |
| `enroll` | `principal: Principal`, `key: Array[u8, 32]` | — | `auth.name-taken` | `recorded` |
| `issue` | `child: Principal` | its public key and private seed, once | — | `none` |
| `rotate` | `principal` and its new key, or `link` or `audit` | — | — | `recorded`; `irreversible{erase}` for `link` and `audit` |
| `revoke` | `principal` | — | — | `recorded` |
| `set_pin` | `principal`, `pin: text` | — | `policy.pin-format` | `restorable` |
| `check_pin` | `principal`, `pin: text` | — | `policy.pin-wrong`, `policy.pin-locked` | `none` |

- `enroll` writes `/sys/principals/<principal>` and `/data/<principal>` ([enrollment](19-oversight.md#enrollment)), and only `link` holds it and `issue`, which serves `delegate` ([delegation](09-authority.md#delegation)).
- `rotate` and `revoke` follow [rotation and revocation](09-authority.md#rotation-and-revocation); `set_pin` and `check_pin` keep the PINs of [console PINs](19-oversight.md#console-pins), and only `console` holds them.
- Its checkpoint holds its keys and PIN tags ([keys on the machine](09-authority.md#keys-on-the-machine)).
- Wiring: the `entropy` role; the `tpm` role, through which it seals its keys and unseals the volume key; ports to `store` and `journal`.

## `journal`

Events, the journal's streams, metrics, traces, flight recorders, replay and the audit chain ([Observability](17-observability.md)), with the operations of [the journal and its streams](17-observability.md#the-journal-and-its-streams). Wiring: the core rings ([core rings](17-observability.md#core-rings)); the `KeyUse` purpose `anchor`; ports to `store` and `keys`.

## `policy`

Decides subjects and keeps approvals ([Oversight](19-oversight.md)). Its configuration record is `PolicyConfig { machine: Array[u8, 16] }`, the machine id ([owner](19-oversight.md#owner)), at `/sys/config/policy`.

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `decide` | the subjects of a plan, `at_apply: bool` | the decision, its rules and the approval requirements | — | `none` |
| `ask` | the plan hash, read-set hash, subjects, facts and intent | the approval id | `policy.ask-limit` | `recorded` |
| `approve` | `approval`, `principal`, a signed `ApprovalDecision` or a console decision, the channel | the approval's state | `auth.bad-signature`, `policy.not-approver`, `policy.decision-mismatch`, `policy.not-pending`, `policy.constraint-refused`, `policy.channel` | `recorded` |
| `withdraw` | `approval` | — | `policy.not-pending` | `recorded` |
| `explain` | `principal`, a subject | each rule it matches and the decision | — | `none` |
| `propose` | `candidate: Policy` | the approval id | `policy.version`, `policy.pattern`, `policy.agent-approver`, `policy.preset-unavailable`, `policy.ask-limit` | `recorded` |

- `link` and `console` hold `decide`, `ask`, `approve` and `withdraw`; `explain` answers any session about its own principal.
- Its checkpoint holds the agent budgets' counters, the first uses and the ask counts.
- Wiring: Views of `/sys/policy`, `/sys/approvals`, `/sys/principals` and `/sys/config/policy`; ports to `keys`, `journal` and `console`.

## `link`

Serves the link protocol on its transports, and `LinkSession` to resident agents ([transports](18-link.md#transports)). It keeps sessions, taints and the extraction ledger ([taint](19-oversight.md#taint)), request keys ([request keys](14-changes.md#request-keys)) and plans ([plans](14-changes.md#plans-and-steps)); its checkpoint holds the principals' taints and the ledger. Its configuration is `LinkConfig` ([configuration](18-link.md#configuration)). Wiring: the `console` role's `read` and `write` for the serial transport; a `Listen` grant for port 7710, through which `net` carries the vsock and TCP transports ([network transports of the link](16-network.md#network-transports-of-the-link)); the `KeyUse` purpose `handshake`; ports to every other base instance.

## `console`

The human's trusted path at the machine. `console` drives the reserved lane of the `console` role ([Console](15-devices.md#console)); it prints the claim ([claim](19-oversight.md#claim)), notices and approvals, takes decisions with presence and PIN, and runs the emergency controls.

- The console is **dedicated** when `ConsoleConfig.dedicated` at `/sys/config/console` is true, by default on a machine with `nic` or a vsock device, and on an unclaimed machine outside `dev` mode; it is **shared** otherwise. A dedicated console keeps a `read_reserved` pending, so every input line reaches `console` and none the link. On a shared console, `link` passes every input line starting `console ` to `console` and treats none as a request ([framing](18-link.md#framing)).
- Presence ([claim](19-oversight.md#claim)) is an answer typed at the console within its prompt's time. `console` writes through `write_reserved`, each output a block with the id `console`, and never prints a PIN.
- A freeze typed at the console commits as `system` with intent `console`.

| Line (after `console ` on a shared console) | Effect |
| --- | --- |
| `yes` | answers the prompt that waits |
| `claim <key>` | on an unclaimed machine, claims it for that Ed25519 public key, presence included ([claim](19-oversight.md#claim)) |
| `status` | prints `/live/status` and `/live/security` ([/live](13-tree.md#live)) |
| `approvals`, `show a<n>` | lists the pending approvals; prints one as [approval records](19-oversight.md#approval-records) renders it |
| `allow a<n> [for <s>]`, `deny a<n>` | reads a principal and its PIN, then decides `Once` or `HashFor(<s>)` ([decisions](19-oversight.md#decisions)) |
| `freeze agents`, `freeze egress`, `freeze all` | the [emergency control](19-oversight.md#emergency-controls) |
| `rollback`, `recovery` | reads the owner's PIN, then the control |
| `pin` | prints a nonce to set a [PIN](19-oversight.md#console-pins) |
| `recover` | in recovery mode, reads the recovery code and the new owner key ([recovery](19-oversight.md#recovery)) |

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `show` | a notice: its kind and record | — | — | `irreversible{emit}` |
| `confirm` | a prompt, `timeout_s: u32` | `confirmed: bool` | — | `irreversible{emit}` |
| `line` | `text::String` | — | — | `none` |
| `pin` | `nonce: Array[u8, 16]` | — | `policy.console-shared`, `policy.request-stale` | `restorable` |
| `freeze` | an `Action`, the principal, and for a request outside a session the `OversightRequest` and its signature | the commit | `auth.bad-signature`, `policy.not-human`, `policy.request-stale` | `restorable` |

`link` holds `confirm`, `line`, `pin` and `freeze`; `policy` holds `show`, for each approval it writes, and `journal`, for each `critical` event. Wiring: the `console` role's `read_reserved` and `write_reserved`; ports to `policy`, `keys`, `boot`, `supervisor`, `store` and `journal`. The claim prints as:

```text
ok console claim: records 1
claim code: gove-paga-fezi-hise-tene-miga-dapi-losa-viho-rele-hale-gove, generation: 7, manifest: af3gp7asackvykf55j, kernel: agcgjamqtp6kwkmmlv, loader: aglsgbpvbmohxyjyml, machine: ts4u6ffjywy24zkd3ptzhnyqemne
ok console claim: records 1
```

## `admission`

Admits, lowers again and removes bundles ([Code](10-code.md)).

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `admit` | `bundle: HashValue`, a blob the store holds | the admission record | the codes of [admission](10-code.md#admission) | `recorded` |
| `lower` | `bundle: HashValue` | — | `code.no-image`, `code.recheck-failed` | `recorded` |
| `remove` | `bundle: HashValue` | — | `code.in-use` | `recorded` |

The plan step `install` runs `admit`, and [toolchain updates](10-code.md#toolchain-updates) run `lower`. Wiring: a View of `/sys/code`; ports to `store`, `keys`, `policy` and `journal`; a `Control` grant with `spawn` to run the `nan` job.

## `boot`

Generations, slots, the floor, owner and loader requests and the health gate ([Boot](03-boot.md)); it mirrors the boot record in `/sys/boot` ([/sys](13-tree.md#sys)).

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `install` | `manifest: Path` | the generation number | the codes of [generation operations](03-boot.md#generation-operations) | `restorable` |
| `rollback` | — | — | `boot.no-fallback` | `restorable` |
| `recovery` | — | — | — | `restorable` |
| `retire` | `generation: u64` | — | `boot.floor-too-high` | `irreversible{erase}` |
| `replace_loader` | `loader: HashValue`, `version: u32`, `signature: Signature` | — | `auth.bad-signature`, `boot.write-failed` | `irreversible{actuate}` |
| `replace_owner` | `key: Array[u8, 32]`, `recovery: Array[u8, 32]`, `signature: Signature`, `manifests: mem::List[Signature]` | — | `auth.bad-signature`, `boot.resign-required`, `boot.write-failed` | `irreversible{actuate}` |
| `claim` | `key: Array[u8, 32]`, `recovery: Array[u8, 32]` | — | `policy.already-claimed`, `policy.unclaimable` | `irreversible{actuate}` |
| `recover` | `code: Array[u8, 16]`, `key: Array[u8, 32]`, `recovery: Array[u8, 32]`, `manifests: mem::List[Signature]` | — | `policy.recovery-code-invalid`, `boot.resign-required`, `boot.write-failed` | `irreversible{actuate}` |
| `set_recovery` | `generation: u64` | — | `boot.not-good` | `restorable` |

- `install` serves the plan step `generation`. `rollback` and `recovery` serve owner sessions and `console` ([emergency controls](19-oversight.md#emergency-controls)); `replace_loader` follows [loader replacement](03-boot.md#loader-replacement), and `replace_owner`, `claim` and `recover` [owner key requests](03-boot.md#owner-key-requests), their `manifests` being the requested key's signatures over the manifests of the generations signed by `owner` that the boot record names, in generation order; `set_recovery` follows [generation operations](03-boot.md#generation-operations). `link` and `console` hold `claim`, and `console` `recover`.
- After a direct boot every operation fails with `boot.no-boot-volume` ([direct boot](03-boot.md#direct-boot)).
- Wiring: ports to `fat`, `power`, `supervisor`, `keys` and `journal`; the `tpm` role.

## `clock`

Wall time ([wall clock](08-scheduling.md#wall-clock)), configured by `ClockConfig { sources: mem::List[text::String] }` at `/sys/config/clock`, the network time sources, empty by default. Each source is the host name of a Network Time Security key establishment server (RFC 8915), which `clock` queries with NTPv4 protected by NTS through its `Egress` grant; a reading's uncertainty is its round-trip time plus the source's root dispersion.

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `set` | `utc_ns: i64` | — | — | `restorable` |
| `sources` | — | each source with its offset, uncertainty and last reading | — | `none` |

Outside an owner session, `set` asks the owner ([evaluation](19-oversight.md#evaluation)). Wiring: the `rtc` role; an `Egress` grant per source.

## `power`

Power off, reboot, and the settings of [frequency](08-scheduling.md#frequency) and [idle states](08-scheduling.md#idle-states).

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `off` | — | — | `fault.power-failed` | `restorable` |
| `reboot` | — | — | `fault.power-failed` | `restorable` |
| `configure` | `config: PowerConfig` | — | `schema.invalid` | `recorded` |

`off` and `reboot` run [power off and reboot](08-scheduling.md#power-off-and-reboot); `configure` writes `/sys/config/power`. Wiring: the kernel policies `power.idle` and `power.perf` ([kernel policies](04-kernel.md#kernel-policies)); a port to `devices.aml`.

## `net`

On a machine with `nic` or a vsock device, the network stack and the egress gate ([Network](16-network.md)), with the operation `configure` of [configuration](16-network.md#configuration) and the capabilities of the [egress gate](16-network.md#egress-gate) and [ingress](16-network.md#ingress). Wiring: the `nic` role; the root `Egress` and `Listen` grants.

## `infer`

Model inference. Its configuration `InferConfig { models: mem::List[Model] }` at `/sys/config/infer` names each model, `Model { name: text::String, server: Moniker, micros_per_1k_tokens: u64 }`, served by an instance implementing the `Infer` protocol of this section: an engine running on the machine, or an adapter that reaches a remote provider through its own `Egress` grant. Every output is `Untrusted`, labelled `model.<name>` ([origin labels](02-identifiers.md#origin-labels)); tokens and money are charged to the calling session ([budgets](18-link.md#budgets), [agent budgets](19-oversight.md#agent-budgets)).

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `models` | — | each model, its server and price | — | `none` |
| `generate` | `model`, `prompt: Bytes`, `max_tokens: u32` | `Untrusted[text::String]` | `budget.exhausted`, the [`net` codes](16-network.md#codes) | `none` for a model whose server holds no `Egress` grant, `irreversible{emit,spend}` otherwise |
| `extract` | `model`, `input: Untrusted[Bytes]`, `schema: HashValue` | the construction text of a value of that schema, `Untrusted` | as `generate` | as `generate` |

`extract` serves [extraction](19-oversight.md#extraction): it asks the model for a value of the type the schema hash names (requires [N20](22-nan.md#n20)). Wiring: a port to each model's server.

## Jobs

A generation ships five jobs, admitted bundles of provenance `vendor` that the plan step `run` or the named operation starts ([jobs](06-execution.md#jobs)):

| Job | Does | Input → result | Started by |
| --- | --- | --- | --- |
| `nan` | the NaN toolchain over a store tree: `check` in verify mode (requires [N26](22-nan.md#n26)), `test`, `build` and lowering; only on a machine with at least 4 GiB of RAM ([derived defaults](01-machines.md#derived-defaults)) | `NanRequest { command, tree, scope }` → `NanReport { verdict, records }` | `admission`, `run` |
| `scrub` | verifies the store's checksums and blob identities ([scrub](12-store-format.md#scrub)) and the whole retained [audit chain](17-observability.md#audit-chain) | → `ScrubReport` | `store`'s `scrub`, `run` |
| `selftest` | checks that every base instance is `converged`, `/live/memory` sums to RAM, the policy record verifies, every grant lies within its parent and every specification's bundle is admitted | → a list of `Check { name, passed, detail }` | `run` |
| `bench` | measures the requirements of [Performance](21-performance.md#measurement) | `BenchRequest` → `BenchReport` | `run` |
| `replay` | re-runs a recorded transition on a fork ([replay](17-observability.md#replay)) | `ReplayRequest` → `ReplayReport` | `journal`'s `replay` |

## Classical equivalents

NaN OS has no executables; link requests ([verbs](18-link.md#verbs)) replace the classical utilities, a `plan` taking effect by `apply <plan>`:

| Classical | Request |
| --- | --- |
| `ps` | `get /live/instances` |
| `top` | `find from /live/accounts order cpu_ns desc limit 10` |
| `kill`, `systemctl stop` | `call supervisor stop(<moniker>)` |
| `systemctl restart` | `call supervisor restart(<moniker>)` |
| `cat`, `ls` | `get <path>` |
| `cp`, `rm` | `plan [write <path> <value>]`, `plan [delete <path>]` |
| `journalctl`, `dmesg` | `find from /journal/log`, `find from /journal/events` |
| `strace` | `watch /live/instances/<moniker>` |
| `gdb` | `history <moniker> --replay <seq>` |
| `apt install` | `plan [install <bundle>, write /sys/instances/<moniker> <spec>]` |
| `git log`, `git revert` | `history <path>`, `undo <commit>` |
| `sudo` | an [approval](19-oversight.md#approvals) |
| `useradd`, `passwd` | `call keys enroll(...)`, `call keys rotate(...)` |
| `chmod`, `chown` | `plan [grant ...]`, `plan [revoke ...]` |
| `crontab` | `apply <plan> --when <time>` |
| `shutdown`, `reboot` | `call power off()`, `call power reboot()` |
| `lspci`, `ip addr` | `get /live/devices`, `get /live/net` |
| `fsck`, `df`, `free` | `call store scrub()`, `call store stats()`, `get /live/memory` |
| `date` | `call clock sources()` |
| `ssh` | a link session over TCP ([network transports of the link](16-network.md#network-transports-of-the-link)) |
