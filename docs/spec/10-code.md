# Code

This chapter defines the units of code, the bundle file with its signatures, certificate and provenance, admission step by step, the admission record, loading and unloading, load-time registries, live update, probation and its revert, rollback after probation, base instances, the generation switch, toolchain updates, and the `code` and `update` code families.

## Artefacts and bundles

The unit of code is a NaN package artefact, named by its code root hash ([NaN 30 §Artefacts](https://github.com/nan-language/nan/blob/main/docs/spec/30-artefacts.md#artefacts)). The unit of loading is a **bundle**: one NaN final closure for `x86_64-nanos` or `aarch64-nanos`, lowered as one image against the support routines the running kernel exports (requires [N1](22-nan.md#n1), [N6](22-nan.md#n6)). Its **root** is its final-link root.

- A **component bundle** exports the components its root's component entries name and its registry contributions, and may hold contributions and no component. A **program bundle** exports its root's `main` and runs as a [job](06-execution.md#jobs).
- Values move only between instances of one bundle loaded in one space; messages between bundles travel encoded ([isolated transport](07-communication.md#isolated-transport)).
- The **code hash** of a bundle is the [hash](02-identifiers.md#hashes) of kind `code` of the `mem::wire` encoding of its `CodeId`, code roots in byte order. Specs, plan steps and `/sys/code/<hash>` name bundles by it, so one closure has one code hash whatever images accompany it.

```nan
pub closed struct CodeId derives Encode {
    pub root: text::String // the root package's name
    pub code_roots: mem::List[HashValue] // every package's code root hash
} CodeId
```

The kernel image's base components form the **base bundle** ([base instances](#base-instances)).

## Bundle file

A **bundle file** is a blob holding the `mem::wire` encoding of a `Bundle`:

```nan
pub closed struct Bundle derives Encode {
    pub root: text::String
    pub artefacts: mem::List[mem::Bytes] // one NaN artefact per package
    pub images: mem::List[Image]
    pub certificate: Option[Certificate]
    pub signatures: mem::List[Signed] // bundle signatures
} Bundle

pub closed struct Image derives Encode {
    pub level: Level
    pub toolchain: HashValue // the toolchain that lowered it
    pub support: HashValue // the support-routine table it imports
    pub bytes: mem::Bytes
} Image

pub closed enum Level derives Copy, Eq, Ord, Render, Encode {
    X86v1
    X86v2
    X86v3
    X86v4
    Armv80
    Armv82
} Level

pub closed struct Signed derives Encode {
    pub principal: Principal
    pub signature: Signature
} Signed
```

- `artefacts` holds one artefact per package of the closure, std included, in byte order of code root hash.
- `images` holds at most one image per level, toolchain and support table. `toolchain` is a [toolchain hash](https://github.com/nan-language/nan/blob/main/docs/spec/36-builds.md#toolchain-hash); `support` is the hash of the support-routine table the image imports, and an image loads only into a kernel exporting a table with that hash, the running kernel's being `LiveBoot.support` ([/live](13-tree.md#live); requires [N6](22-nan.md#n6)).
- `X86v1` to `X86v4` are the x86-64 microarchitecture levels 1 to 4 of the System V psABI, `Armv80` is Armv8.0-A and `Armv82` Armv8.2-A with the dot product instructions. A machine's level is `Machine.level` ([measured facts](01-machines.md#measured-facts)); an image runs on a machine of its level or above (requires [N17](22-nan.md#n17)).
- The **bundle hash** is the hash of kind `bundle` of the bundle's encoding with `signatures` empty; each entry of `signatures` is a `bundle` signature ([signatures](09-authority.md#signatures)).

### Signatures and provenance

These parts of a bundle's artefacts carry signed facts that provenance and admission read (requires [N15](22-nan.md#n15)):

| Part | Holds |
| --- | --- |
| an artefact's signature section ([NaN 30 §Reserved sections](https://github.com/nan-language/nan/blob/main/docs/spec/30-artefacts.md#reserved-sections)) | `review` signatures, each with its principal; the object of kind `review` is the 33-byte code root hash |
| an artefact's provenance section | the principals who are the package's **authors** |
| the signature field of each seal record of an artefact's attestation file ([NaN 31 §Seals](https://github.com/nan-language/nan/blob/main/docs/spec/31-trust.md#seals)) | a `seal` signature by the seal's principal; the object of kind `seal` is the seal record's encoding without its signature field |

A bundle's `certificate` is a **check certificate**, part of NaN OS's bundle format:

```nan
pub closed struct Certificate derives Encode {
    pub builder: Principal
    pub code: HashValue // the code hash checked
    pub toolchain: HashValue // the toolchain that checked it
    pub images: mem::List[HashValue] // blob ids of the images it lowered
    pub signature: Signature
} Certificate
```

It states that `builder` checked the closure of code hash `code` in verify mode with `toolchain` and found no error, and lowered the listed images. Its `signature` is a `certificate` signature; the object of kind `certificate` is the certificate's encoding with `signature` set to zero bytes. A seal is accepted when it is current ([NaN 31 §Seals](https://github.com/nan-language/nan/blob/main/docs/spec/31-trust.md#seals)) and its signature verifies. An image of the bundle file is **trusted** when the bundle carries a `bundle` signature by a `vendor.*` principal, or a certificate by a builder the [policy](19-oversight.md#policy) trusts lists the image's blob id.

## Admission

**Admission** is the plan step `install <bundle blob path or blob id>` ([steps](14-changes.md#steps)), which a plan's principal may take while it holds an `Instance` grant with `spawn` or `update` (`auth.denied`). The `admission` instance runs these steps during the preview, for the bundle and for the activations of the plan; each failure refuses the plan:

1. **Decode.** The blob decodes as the canonical encoding of a `Bundle` and each artefact decodes (`code.malformed`). The root package declares the nanos target of the machine's architecture (`code.target-mismatch`).
2. **Closure.** The bundle holds exactly one artefact per package name, and every `dep` directive of every package resolves to the artefact of that package's name (`code.closure-incomplete`).
3. **Signatures.** Every signature of the bundle, its certificate, its signature sections and its seals verifies against a principal record as [signatures](09-authority.md#signatures) states (`auth.bad-signature`).
4. **Check.** On a machine running the [`nan` job](20-base.md#jobs), admission checks the closure in verify mode: well-formedness, every obligation decided anew and every example run, whatever the requires-sets (requires [N26](22-nan.md#n26)); an error fails with `code.check-failed`, the NaN diagnostic in its because chain. On any other machine, the bundle needs a certificate for its code hash by a builder the policy trusts (`code.unsigned`).
5. **Requires-sets.** Each requires-set entry naming an item of the platform package or a kernel-implemented std item is satisfied, as [NaN 29 §Requires-sets](https://github.com/nan-language/nan/blob/main/docs/spec/29-identity-versioning.md#requires-sets) defines, when the running kernel image's releases replace the bundle's (`code.requires-unsatisfied`).
6. **Seals.** Every trust escape of the closure outside the std packages of the toolchain of step 4 carries an accepted seal by one of the policy's `sealers` ([policy](19-oversight.md#policy)) (`code.seal-missing`). A `device` or `priv` grant needs one by a `vendor.*`, `human.*` or `owner` principal, except a `device` grant confined to IOMMU domains on a machine with `iommu` (`code.seal-rejected`). No accepted seal is by an author of the sealed item's package or of a package holding an item of its closure entries (`code.seal-same-author`).
7. **Authority.** Admission computes the closure summary ([NaN 14 §Authority summaries](https://github.com/nan-language/nan/blob/main/docs/spec/14-authority.md#authority-summaries)), whose entries include the OS capability kinds (requires [N27](22-nan.md#n27)). The [effect set](14-changes.md#the-effect-set) lists it, and for an `update`, as widenings, the entries the running bundle's summary lacks; the [policy](19-oversight.md#policy) decides the approvals they need.
8. **Schemas.** For each instance an `update` step activates, the new component's state schema equals the running one, accepts its schema tree ([NaN 29 §Encode class](https://github.com/nan-language/nan/blob/main/docs/spec/29-identity-versioning.md#encode-class)), or is covered by a `migrate` of the new bundle from the running schema hash (requires [N5](22-nan.md#n5), [N20](22-nan.md#n20)); otherwise `code.schema-incompatible`. The step's **migration class** is `identical`, `additive` or `migrated` accordingly.
9. **Protocols.** Every port wired to or from an activated instance joins compatible protocols ([protocols](07-communication.md#protocols); `port.incompatible`).
10. **Reversibility.** Admission records the declared reversibility bound of each operation each component serves, the bundle's **reversibility profile**. Its checked digest respects each bound, except a bound lower than the class of the operations it calls, which only a sealed premise declares, a trust escape that step 6 checks ([declared classes](14-changes.md#declared-classes); requires [N16](22-nan.md#n16)).
11. **Contributions.** Each contribution to a registry of the kernel image is admitted as [registries](#registries) states.
12. **Provenance and placement.** Admission computes the provenance class and the `shared` field ([provenance](09-authority.md#provenance), [placement](09-authority.md#placement)); a step starting an instance whose placement the machine cannot give is refused with its `place` code.
13. **Lowering.** Admission takes a trusted image of the machine's level whose `support` is the running kernel's; else, with the `nan` job, it lowers the closure for that level (requires [N6](22-nan.md#n6), [N17](22-nan.md#n17)); else it takes such an image of the highest lower level; else it fails with `code.no-image`, whose `expected:` line gives `Machine.level` and `LiveBoot.support`.
14. **Record.** The apply writes the [admission record](#admission-record), the bundle blob and the image in the plan's commit, and emits `code.admitted`.

Admitting a code hash already admitted runs steps 7 to 12 only. Admission is charged to the session's [account](08-scheduling.md#accounting), and results of verify mode and lowering are reused across previews of one bundle.

## Admission record

`/sys/code/<hash>` is a dir holding `admission`, the admission record, `bundle`, the bundle blob, and `image`, the image the machine loads. Only `admission` writes it, and at a generation's initial boot it writes the base bundle's record from the genesis record.

```nan
pub closed struct Admission derives Encode {
    pub root: text::String
    pub packages: mem::List[Package]
    pub components: mem::List[Component]
    pub program: Option[Program]
    pub authority: mem::List[text::String] // closure summary entries
    pub escapes: mem::List[Escape]
    pub contributions: mem::List[Contribution]
    pub provenance: Provenance
    pub shared: bool // placement conditions 1 and 2 hold
    pub check: CodeCheck
    pub image: Lowered
    pub admitted: CommitId
} Admission

pub closed struct Package derives Encode {
    pub name: text::String
    pub code_root: HashValue
    pub authors: mem::List[Principal]
    pub reviewers: mem::List[Principal] // principals of verified review signatures
} Package

pub closed struct Component derives Encode {
    pub state: text::String // canonical path of the state type
    pub schema: HashValue // state schema
    pub protocol: HashValue // protocol hash of its messages
    pub operations: mem::List[Operation]
    pub env: mem::List[Field]
    pub config: Option[text::String] // canonical path of its configuration type
} Component

pub closed struct Operation derives Encode {
    pub name: text::String
    pub digest: text::String // NaN's digest line
    pub reversibility: text::String // its declared bound
} Operation

pub closed struct Field derives Encode {
    pub name: text::String
    pub kind: text::String // canonical path of the capability type
    pub protocol: Option[HashValue] // for a port
    pub optional: bool
} Field

pub closed struct Program derives Encode {
    pub input: Option[HashValue] // schema hashes of a typed main
    pub output: Option[HashValue]
} Program

pub closed struct Escape derives Encode {
    pub item: text::String
    pub kind: text::String // ledger kind
    pub sealers: mem::List[Principal]
} Escape

pub closed struct Contribution derives Encode {
    pub table: text::String // the registry's canonical path
    pub key: text::String // the rendered key
    pub replaces: bool
} Contribution

pub closed enum CodeCheck derives Encode {
    Verified(toolchain: HashValue)
    Certified(builder: Principal, toolchain: HashValue)
} CodeCheck

pub closed struct Lowered derives Encode {
    pub level: Level
    pub toolchain: HashValue
    pub support: HashValue
    pub image: HashValue // blob id
    pub code_bytes: u64
} Lowered
```

`Provenance` is the type of [provenance](09-authority.md#provenance), and the other named types are those of [identifiers](02-identifiers.md#types). `admission` rewrites `image` when it lowers again, and `provenance` and `shared` when it recomputes them ([provenance](09-authority.md#provenance)). The revocation of a key whose signature an admission relied on runs admission again for that bundle ([rotation and revocation](09-authority.md#rotation-and-revocation)); when it refuses the bundle, its instances stop and enter `failed` with the refusal's code. The operation `remove` of `admission` ([admission](20-base.md#admission)) deletes the dir; it fails with `code.in-use` while a spec or a job names the code hash.

## Loading

The runner **loads** a bundle into a space when it starts an instance of the bundle there and no other instance there uses it ([starting](06-execution.md#starting)) (requires [N6](22-nan.md#n6)):

1. It places the image's code in a code area of the space ([address spaces](04-kernel.md#address-spaces)) and maps its read-only data read only and its writable data read-write and non-executable. The code and read-only frames of an image are shared by every space that loads it.
2. It binds the image's support-routine imports to the kernel's exports; an image whose `support` differs fails with `code.no-image`.
3. It adds the bundle's registry contributions ([registries](#registries)) and emits `code.loaded`.

The frames of a loaded bundle are charged to the memory budget ([cells and limits](05-memory.md#cells-and-limits)) of the instance whose start loaded it into the space, and pass to another instance using it there when that one stops. When no instance in a space uses the bundle, the runner removes its contributions, unmaps it and emits `code.unloaded`; its frames are freed once no space maps them. The time to start an instance or a job from an admitted bundle is bounded in [Performance](21-performance.md).

## Registries

The registries of the kernel image are **load-time registries**: their tables at the kernel image's final build ([NaN 25 §Tables](https://github.com/nan-language/nan/blob/main/docs/spec/25-registries.md#tables)) are their initial entries, and loading a bundle adds its contributions (requires [N6](22-nan.md#n6)).

- A bundle contributes to a kernel registry through a contribution its root admits ([NaN 25 §Contributors](https://github.com/nan-language/nan/blob/main/docs/spec/25-registries.md#contributors)), declared in a release of the registry's package compatible with the kernel image's.
- A contribution whose key equals a live entry's is refused (`code.registry-duplicate`), except to a kernel policy registry, where it replaces the active entry as [kernel policies](04-kernel.md#kernel-policies) states; the effect set lists each replacement.
- The kernel policy registries `sched.pick`, `sched.place`, `mem.reclaim`, `power.idle` and `power.perf` take contributions only from bundles of provenance `vendor` or `reviewed` whose `shared` field holds (`code.registry-denied`).
- Loading replaces a table atomically: a reader of the previous table finishes with it. A function field of a contributed entry is called through N6's convention across closures.
- Unloading removes the bundle's entries and restores those they replaced.

## Live update

The plan step `update <moniker> <hash>` writes the spec's `bundle` ([instance specifications](06-execution.md#instance-specifications)); its preview admits the bundle when needed and runs admission steps 8 and 9. A step updating an instance whose previous update is in probation is refused (`update.in-probation`). A step updating a checkpointed base instance is refused (`code.schema-incompatible`) unless the new state schema is accepted by the base bundle of the running generation, of the recovery generation and of the other slot's generation when that slot is `good`. Reconciliation then updates the running instance:

1. The runner loads the new bundle into the instance's space, or into the space its new placement gives; a loading failure aborts the update with its code.
2. It pauses the intake, so messages queue, waits for the running handler call to return and cancels the keyed tasks with acknowledgement. Exceeding the plan's activation timeout ([concurrency](14-changes.md#concurrency)) aborts the update (`plan.activation-timeout`).
3. It encodes the state with the old bundle into memory, the **pre-update encoding**; a dirty checkpointed instance also checkpoints ([checkpoints](06-execution.md#checkpoints)).
4. It decodes the encoding with the new bundle, through its `migrate` for class `migrated` (requires [N5](22-nan.md#n5)). The state is re-encoded and decoded even when the schema is unchanged. A decoding failure aborts the update (`update.decode-failed`).
5. It builds the new Env ([Env construction](09-authority.md#env-construction)); a failure aborts the update with its code.
6. It swaps the bundle; the runner tables follow [runner tables](06-execution.md#runner-tables), and the queued messages [transfer](07-communication.md#transfer).
7. It calls `resume` with reason `Updated`, reopens the intake, emits `update.started`, and starts probation.

An aborted update leaves the instance `running` its old bundle. When the abort follows step 3, the old code decodes the pre-update encoding; when it follows the cancellation of the keyed tasks in step 2, the runner calls `resume` with reason `Updated`; the intake then reopens. The spec's `bundle` returns to the old code hash: within the update's [health window](14-changes.md#health-windows), by its revert by blame; otherwise, by a revert commit of `supervisor`. The runner emits `update.reverted` with the abort's code as its reason.

The unavailability of steps 2 to 7, without the acknowledgement of cancelled I/O, is bounded in [Performance](21-performance.md). A `hibernated` instance is revived to update; a `stopped` one runs the new bundle at its next start.

## Probation

**Probation** of an update ends when 60 s (`Probation.min_s`) have passed and the instance has handled 100 messages (`Probation.min_messages`), or after 600 s (`Probation.max_s`). Instances updated by one commit share one probation, so a revert of one reverts them all. During it:

- the old bundle stays loaded, the pre-update encoding stays in memory, and the checkpoint taken before the update stays kept ([checkpoints](06-execution.md#checkpoints));
- the [flight recorder](17-observability.md#flight-recorders) records the instance's full inputs and the results of its nondeterministic capability calls;
- the runner asks `Health` at its start and every 10 s (`Probation.health_s`), as [system interfaces](06-execution.md#system-interfaces) states.

It reverts the update on an abandonment of the instance's domain ([trap handling](06-execution.md#trap-handling)), on a failed health check, or on a **regression**: at its end, the p99 handler latency during probation exceeds twice (`Probation.latency_factor`) the previous bundle's p99 over its last hour of running, plus 100 µs (`Probation.latency_margin_us`). No regression is computed when the previous bundle handled fewer than 100 messages in that hour. A probation also ends, without a revert, when its recording reaches 16 MiB (`Probation.max_record_bytes`), at a stop or a restart of an instance it covers, and at a reboot. A probation ending without a revert emits `update.passed` unless a reboot ended it. The old bundle then unloads when unused. These fields form the record `Probation`, the field `probation` of `SupervisorConfig` at `/sys/config/supervisor` ([`supervisor`](20-base.md#supervisor)).

A **revert** runs these steps:

1. The runner pauses the intake, waits for the running handler call and cancels the keyed tasks with acknowledgement.
2. It discards the new state, decodes the pre-update encoding with the old bundle and builds the old Env.
3. It calls the old code's handler on each recorded input, in order, releasing none of the held effects of these calls. When a call traps, its input becomes a [dead letter](06-execution.md#dead-letters), and the replay starts again from the pre-update encoding without it.
4. It calls `resume` with reason `Updated` and reopens the intake.
5. The spec's `bundle` returns to the old code hash: during the update's health window, by its revert by blame ([health windows](14-changes.md#health-windows)); after it, by a revert commit of `supervisor`. The runner emits `update.reverted`.

## Rollback after probation

`undo` of an update commit ([undo](14-changes.md#undo)) previews an `update` back to the old code hash. That update is lossless when the schema hash is unchanged, when the new schema is additive over the old one and the old state type is `flexible`, so the old decoder keeps the new fields, or when the `migrate` declares an inverse (requires [N5](22-nan.md#n5)). Otherwise the preview offers `restore-checkpoint <moniker> <n>` with the checkpoint taken before the update while the store keeps it ([checkpoints](06-execution.md#checkpoints)), else a restart, and lists the commits the instance has written since the update, whose effects the restored state no longer reflects.

## Base instances

The base bundle has provenance `vendor` and its instances act for `system`. Its admission record is part of the generation's genesis record ([genesis](13-tree.md#genesis)). A base instance updates like any instance; the updated instance then exchanges encoded messages with the other base instances.

## Generation switch

A new generation's kernel image folds in the bundles of updated base instances, each of which must have the placement of its base instance for `system`; its manifest is signed as [generations and manifests](03-boot.md#generations-and-manifests) states. The preview of the plan step `generation <manifest>` refuses the switch when the state schema of a checkpointed base instance is not accepted by the new base bundle, or when the new base bundle's state schema of a checkpointed base instance is not accepted by the base bundle of the recovery generation or of a `good` generation in the slot other than the idle slot ([generation operations](03-boot.md#generation-operations)) (`code.schema-incompatible`), when a port between instances joins protocols the new generation makes incompatible (`port.incompatible`), or when a bundle an instance uses has no trusted image for the new kernel's support table (`code.no-image`), unless the new generation ships the `nan` job on a machine running it. Its apply:

1. `boot` installs the generation ([generation operations](03-boot.md#generation-operations)); a failure ends the plan with its `boot` code.
2. `power` reboots the machine ([power off and reboot](08-scheduling.md#power-off-and-reboot)). When the reboot fails with `fault.power-failed`, `next once` still names the new generation's slot, and the next boot starts it.
3. The new generation runs on the branch `trial/<n>` ([trial branches](11-store.md#trial-branches)), and the [health gate](03-boot.md#health-gate) decides it; a failing generation gives way to the previous one. Admission lowers the images it lacks before their instances start, and each instance restores from its checkpoint under the new code as [starting](06-execution.md#starting) states.

## Toolchain updates

A **toolchain update** installs a new bundle of the `nan` program and makes it the one the `nan` job runs ([jobs](20-base.md#jobs)). After it, `admission` checks again in verify mode and lowers again, in CPU class `background` ([CPU classes](08-scheduling.md#cpu-classes)), every admitted bundle an instance uses, against the running kernel's support routines. Each bundle lowered emits `code.relowered`, gains the image in its record, and its instances update to the same code hash with the new image: an update of class `identical`, with probation. A bundle failing the check keeps its image and emits `code.recheck-failed`.

## Events

| Kind | Payload | Section |
| --- | --- | --- |
| `code.admitted` | code hash, root, provenance, `shared`, check | [Admission](#admission) |
| `code.loaded` | code hash, space, code bytes | [Loading](#loading) |
| `code.unloaded` | code hash, space | [Loading](#loading) |
| `code.relowered` | code hash, toolchain, level | [Toolchain updates](#toolchain-updates) |
| `code.recheck-failed` | code hash, toolchain, diagnostic | [Toolchain updates](#toolchain-updates) |
| `update.started` | moniker, old and new code hash, migration class | [Live update](#live-update) |
| `update.passed` | moniker, code hash, messages handled, what ended it: its bounds, the recording bound, a stop or a restart | [Probation](#probation) |
| `update.reverted` | moniker, old and new code hash, reason, inputs replayed, inputs dead-lettered | [Live update](#live-update), [Probation](#probation) |

## Codes

| Code | Meaning | Section |
| --- | --- | --- |
| `code.malformed` | the bundle or one of its artefacts does not decode canonically | [Admission](#admission) |
| `code.target-mismatch` | the root declares no nanos target of the machine's architecture | [Admission](#admission) |
| `code.closure-incomplete` | a package is missing or present twice, or a `dep` resolves to no artefact | [Admission](#admission) |
| `code.check-failed` | verify mode found an error | [Admission](#admission) |
| `code.unsigned` | no toolchain and no certificate by a trusted builder | [Admission](#admission) |
| `code.requires-unsatisfied` | the kernel's releases do not satisfy a requires-set | [Admission](#admission) |
| `code.seal-missing` | a trust escape has no accepted seal by an accepted sealer | [Admission](#admission) |
| `code.seal-rejected` | a `device` or `priv` grant sealed by neither a vendor nor a human | [Admission](#admission) |
| `code.seal-same-author` | a seal by an author of what it seals | [Admission](#admission) |
| `code.schema-incompatible` | a state schema neither accepted nor covered by `migrate` | [Admission](#admission) |
| `code.no-image` | no trusted image and no toolchain to lower one, or an image for another support table | [Admission](#admission) |
| `code.in-use` | removing a bundle a spec or a job names | [Admission record](#admission-record) |
| `code.registry-duplicate` | a contribution's key is a live entry's | [Registries](#registries) |
| `code.registry-denied` | a kernel policy contributed by a bundle of provenance `agent` or not `shared` | [Registries](#registries) |
| `update.in-probation` | updating an instance whose previous update is in probation | [Live update](#live-update) |
| `update.decode-failed` | the new bundle does not decode the pre-update encoding | [Live update](#live-update) |
