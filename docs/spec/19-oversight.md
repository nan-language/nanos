# Oversight

This chapter defines the owner, the claim and recovery, enrollment, the policy and its presets, policy proposals, approvals, taint, the rule of two, extraction, agent budgets, the emergency controls, and the `policy` and `taint` code families.

## Owner

The **owner** is the principal `owner` ([principals](09-authority.md#principals)), holder of the grant the claim writes. The **owner key**, which verifies every `owner` signature, is the key the loader stores and reports in the handoff record ([owner key storage](03-boot.md#floor-and-owner-key-storage)). At each boot, when `/sys/principals/owner` holds another key, `keys` rewrites it to the loader's and `policy.owner-mismatch` is emitted.

The **machine id** is 16 bytes that `policy` draws from the kernel's entropy when it first starts after [genesis](13-tree.md#genesis), and keeps in `/sys/config/policy` (`PolicyConfig.machine`). A plan changing it is refused (`policy.machine`). Signatures of the kinds `policy`, `approval` and `request` ([hashes](02-identifiers.md#hashes)) cover the `mem::wire` encodings of `Policy`, `ApprovalDecision` and `OversightRequest`, which name the machine id, and of the boot volume's `OwnerKeyRequest` and `LoaderRequest`, which the loader verifies without it ([owner key requests](03-boot.md#owner-key-requests)).

## Claim

A machine is **unclaimed** while the loader stores no owner key and no claim has committed (step 4). Outside `dev` mode ([modes](03-boot.md#modes)) its link serves only `status`, `primer` and `hello --claim` ([connections and sessions](18-link.md#connections-and-sessions)). The claim runs these steps:

1. At each boot of an unclaimed machine, after each failed claim and 10 min after the last code, the kernel draws a 128-bit **claim code** from its entropy. `console` prints it with the generation, the short forms of the manifest, kernel image and loader hashes ([handoff record](03-boot.md#handoff-record)), and the machine id.
2. The claimant sends `hello --claim <code> <key>` over vsock or TCP, or types `claim <key>` at the console, dedicated while unclaimed outside `dev` mode ([`console`](20-base.md#console)), skipping step 3. `<key>` is an Ed25519 public key; `--encrypt` encrypts the store on a machine with `tpm` ([volumes](12-store-format.md#volumes)). A wrong or expired code fails with `policy.claim-code`, a third in a row drawing a new code; on a claimed machine the request fails with `policy.already-claimed`, and on a machine whose handoff record has flags 4 and 5 clear with `policy.unclaimable` ([floor and owner key storage](03-boot.md#floor-and-owner-key-storage)).
3. `console` prints the key's fingerprint, its hash's short form, and the transport, and waits 120 s for **presence**: the answer `yes` at the console, skipped in `dev` mode. Any other outcome fails the claim with `policy.claim-declined`.
4. The kernel draws the 128-bit **recovery code**. `boot` writes the key and the code's BLAKE3-256 digest as the claim request ([owner key requests](03-boot.md#owner-key-requests)), and `keys` commits the owner's principal record, `/data/owner` and the owner's grant ([grants](09-authority.md#grants)). The machine counts as claimed from this commit, although the loader stores the key only at the next boot; until then the claimed key verifies `owner` signatures.
5. `console` prints the recovery code once; the loader keeps only its digest.
6. On a dedicated console, `console` offers to set the owner's [PIN](#console-pins).
7. `policy.claimed` is emitted; `hello --claim` opens the owner's session.

### Code words

Claim and recovery codes print as twelve code words. A code's 128 bits, followed by the first 16 bits of their BLAKE3-256 hash, form 144 bits, cut from the most significant bit into twelve 12-bit values. Value v is the word `C[v >> 8] V[(v >> 6) & 3] C[(v >> 2) & 15] V[v & 3]`, where `C` is `bdfghjklmnprstvz` and `V` is `aeio`, indexed from 0: 0 is `baba`, 4,095 is `zozo`. Printed codes and requests join the words with `-`, console input with spaces; input ignores case, and failing check bits make a wrong code.

### Recovery

In recovery mode ([recovery](03-boot.md#recovery)), a human types at the console `recover`, the recovery code and a new owner key. `boot` checks the code against the loader's digest (`policy.recovery-code-invalid`) and writes the replacement request with the new key and a new code's digest; `console` prints the new code once, and `policy.recovered` is emitted. The new key takes effect at the next boot.

## Enrollment

**Enrollment** adds a `human.*`, `agent.*`, `builder.*` or `vendor.*` principal: a plan whose step `call keys enroll(...)` writes its principal record and `/data/<principal>` ([`keys`](20-base.md#keys)), with `grant` steps for its grants ([grants](09-authority.md#grants)). The owner enrolls every kind; a principal holding a `Principals` grant with `enroll` enrolls under its prefix, within its own grants. A `grant` step conferring `enroll` on a principal of an agent's family is refused (`policy.agent-enroll`): agents create principals only by [delegation](09-authority.md#delegation).

A principal's **family** is the enrolled principal it is or descends from, all its descendants, and `inst.<moniker>` of each instance [acting for](09-authority.md#principals) one of them; `inst.<moniker>` has the family of the principal its instance acts for.

## Policy

The **policy** decides session requests. `/sys/policy` holds the `SignedPolicy` [`policy`](20-base.md#policy) applies; `Provenance` is the enum of [provenance classes](09-authority.md#provenance):

```nan
pub closed enum Decision derives Encode {
    Allow
    Notify
    Ask
    Deny
} Decision

pub closed enum TaintTest derives Encode {
    Any
    Empty
    Tainted
} TaintTest

pub closed enum DestinationTest derives Encode {
    Any
    Allowlisted
    Other
} DestinationTest

pub closed enum Channel derives Encode {
    Any
    Independent
} Channel

pub closed enum BudgetKind derives Encode {
    NoReturn
    Egress
    Energy
    Tokens
    Money
} BudgetKind

pub closed struct Window derives Encode {
    pub days: u8
    pub from_min: u16 where from_min < 1_440
    pub to_min: u16 where to_min <= 1_440
} Window

pub closed struct Rule derives Encode {
    pub name: text::String
    pub decision: Decision
    pub principals: mem::List[text::String]
    pub operations: mem::List[text::String]
    pub objects: mem::List[text::String]
    pub taint: TaintTest
    pub destinations: DestinationTest
    pub rule_of_two: bool
    pub budget_above: mem::List[BudgetKind]
    pub provenance: mem::List[Provenance]
    pub window: Option[Window]
    pub first_use: bool
    pub approvers: mem::List[text::String]
    pub quorum: u8 where quorum >= 1
    pub expiry_min: u16 where expiry_min >= 1 and expiry_min <= 1_440
    pub channel: Channel
} Rule

pub closed struct AgentBudget derives Encode {
    pub principals: text::String
    pub no_return_per_hour: u32
    pub egress_bytes_per_day: u64
    pub energy_mj_per_day: u64
    pub tokens_per_day: u64
    pub money_per_day: u64
} AgentBudget

pub closed struct Policy derives Encode {
    pub version: u64
    pub machine: Array[u8, 16]
    pub preset: text::String
    pub rules: mem::List[Rule]
    pub allowlist: mem::List[text::String]
    pub sensitive: mem::List[text::String]
    pub reviewers: mem::List[text::String]
    pub sealers: mem::List[text::String]
    pub builders: mem::List[text::String]
    pub agent_shared_space: bool
    pub asks_pending: u16
    pub asks_per_hour: u16
    pub extraction_bits: u16
    pub budgets: mem::List[AgentBudget]
} Policy

pub closed struct SignedPolicy derives Encode {
    pub policy: Policy
    pub signature: Option[Signature]
} SignedPolicy
```

- `preset` is `locked`, `supervised`, `autonomous` or `custom`; `Window.days` has bit 0 for Monday to bit 6 for Sunday, with UTC minutes; `allowlist` holds destination patterns, `sensitive` path globs.
- `reviewers` names the reviewers of [provenance](09-authority.md#provenance), `sealers` the signers of seals admission accepts, `builders` the signers of check certificates it trusts ([admission](10-code.md#admission)); `agent_shared_space` admits agent code without trust escapes to the shared space ([placement](09-authority.md#placement)).
- A rule field the [presets](#presets) leave blank is `Any`, empty, `false` or absent; `approvers` is `owner` and `human.**`, `quorum` 1, `expiry_min` 15.

### Patterns

```ebnf
operation = "*" | "step." ( STEP | "*" ) | "op." moniker_glob "." ( NAME | "*" )
    | "reversibility." ( "none" | "recorded" | "restorable" | "*" )
    | "irreversible" ( "." ( "emit" | "actuate" | "erase" | "spend" ) )?
    | "widening" | "trust-escape" | "downgrade" | "delegate" | "extract" ;
object = "*" | "@base" | path_glob | destination | moniker_glob ;
principal_glob = moniker_glob ;
moniker_glob = glob_segment ( "." glob_segment )* ;
path_glob = ( "/" glob_segment )+ ;
destination = host ":" ( PORT | PORT "-" PORT | "*" ) ;
host = moniker_glob | ADDRESS "/" PREFIX ;
glob_segment = "*" | "**" | SEGMENT ;
```

`SEGMENT` is a name or path segment ([monikers](02-identifiers.md#monikers), [paths](02-identifiers.md#paths)), `STEP` a step kind ([steps](14-changes.md#steps)), `NAME` an operation, `PORT` a decimal port, `ADDRESS "/" PREFIX` an IP prefix. `*` matches one segment, `**` any number. `@base` matches base instances' monikers, their paths `/sys/instances/<moniker>` and `/sys/config/<moniker>`, and `/sys/config/auth`. A pattern matches only objects of its form, and `*` alone every subject.

### Subjects

A **subject** is what the policy decides: a step of a plan or of a `call`, or a `delegate` or `extract` request ([verbs](18-link.md#verbs)). It has its session's principal, for a session of `inst.<moniker>` its instance's `InstanceSpec.principal` ([instance specifications](06-execution.md#instance-specifications)), and:

- operation names: `step.<kind>`; `op.<moniker>.<operation>` for a `call`; `reversibility.<class>`, or `irreversible` and `irreversible.<kind>` per kind, from its preview ([the effect set](14-changes.md#the-effect-set)); `widening` when its effect set widens authority; `trust-escape` for an `install` whose closure holds one; `downgrade` for a `generation` step whose manifest's generation does not exceed the running generation; `delegate`; `extract`, or, for an extraction through a model whose `extract` operation is irreversible ([`infer`](20-base.md#infer)), `irreversible` and `irreversible.<kind>` per kind of that operation's class in its place;
- objects: the paths its preview writes, `<host>:<port>` of each outbox entry it adds, the moniker a `start`, `stop`, `update`, `restore-checkpoint`, `redeliver`, `call` or `run` step names, the child of a `delegate`, the input of an `extract` and, for an extraction through a model, the model's server moniker;
- a provenance: the bundle's class for `install` and `update`; for `generation`, `vendor` or `reviewed` as a `vendor.*` key or the owner signs the manifest.

A `put` is decided as a `write` of its path. `Ask` on a `delegate` or `extract` request counts as `Deny`.

### Evaluation

`policy` decides each subject at preview and again at apply ([apply](14-changes.md#apply)):

1. A subject outside its session's grants fails with `auth.denied` ([sessions](09-authority.md#sessions-as-authority)) before any decision.
2. **Fixed rules** hold whatever the record says: a write of a record [validity](#validity) accepts is allowed; a step using a grant with the `ask` caveat asks ([caveats](09-authority.md#caveats)); `call keys rotate(...)`, `call keys revoke(...)` and `call clock set(...)` outside an owner session ask the owner.
3. A rule **matches** a subject when globs of `principals`, `operations` and `objects` match its principal, one of its operation names and one of its objects, and each condition holds:
   - `taint`: `Empty` when the session's [taint](#taint) is empty, `Tainted` otherwise;
   - `destinations`: `Allowlisted` when the subject adds outbox entries, each leaving through the egress gate to a destination matching `allowlist`; `Other` when one does not;
   - `rule_of_two`: the plan meets the [rule of two](#rule-of-two); `budget_above`: it exceeds a listed [agent budget](#agent-budgets); `provenance`: the subject has a listed class;
   - `window`: the wall time lies in it, and while the clock's uncertainty exceeds 1 min it holds for `Ask` and `Deny` rules only;
   - `first_use`: the principal's family has had no subject with the same operation names and objects allowed under this rule.
4. A subject's decision is the strongest of its matching and fixed rules', `Deny` over `Ask` over `Notify` over `Allow`, and `Deny` without one.
5. A plan's decision is the strongest of its subjects'. `Deny` refuses it (`policy.denied`), naming the rule or `no rule`; `Ask` requires one [approval](#approvals) meeting every asking rule; `Notify` proceeds and emits `policy.notified`. An apply decided more strongly than its preview asks or is refused accordingly.

### Validity

`policy` accepts a `SignedPolicy`, at each write and each boot, only when:

1. `signature` verifies against the owner key for the purpose `policy` ([signatures](09-authority.md#signatures)), or the record is the policy of the generation's genesis record ([genesis](13-tree.md#genesis)) (`auth.bad-signature`);
2. a written record's `version` exceeds the accepted one's (`policy.version`), and `machine` is the machine id, the genesis policy excepted (`policy.machine`);
3. every pattern parses (`policy.pattern`);
4. no glob of `approvers`, `reviewers` or `sealers` can match a principal starting with `agent.` or `inst.` (`policy.agent-approver`);
5. on a machine without `iommu`, `preset` is not `autonomous` and `agent_shared_space` is false (`policy.preset-unavailable`).

A plan writing a failing record is refused with its code. A record failing at boot makes `policy` decide by `locked`, emitting `policy.invalid`, up to the next accepted record. In `dev` mode the owner session's records need no signature. Each accepted record emits `policy.changed`.

## Presets

Rules 1 and 2 name the principals `owner` and `human.**`, the others `agent.**`; "independent" sets `channel` to `Independent`.

| # | Rule | Decision | Operations | Objects | Conditions | Presets |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | `owner` | allow | `*` | `*` | | L S A |
| 2 | `humans` | allow | `*` | `*` | | L S A |
| 3 | `agent-requests` | allow | `delegate`, `extract` | `*` | | L S A |
| 4 | `agent-steps` | allow | `reversibility.*` | `*` | | S A |
| 5 | `no-return` | ask | `irreversible` | `*` | independent | S |
| 6 | `no-return-held` | ask | `irreversible.actuate`, `irreversible.erase`, `irreversible.spend` | `*` | independent | A |
| 7 | `egress-allowlisted` | allow | `irreversible.emit` | `*` | taint `Empty`; destinations `Allowlisted` | A |
| 8 | `egress-other` | ask | `irreversible.emit` | `*` | destinations `Other`; independent | A |
| 9 | `egress-tainted` | ask | `irreversible.emit` | `*` | taint `Tainted`; independent | A |
| 10 | `widenings` | ask | `widening` | `*` | | S A |
| 11 | `trust-escapes` | ask | `trust-escape` | `*` | | S |
| 12 | `trust-escapes-unvendored` | ask | `trust-escape` | `*` | provenance `Reviewed`, `Agent` | A |
| 13 | `generations` | ask | `step.generation` | `*` | | S |
| 14 | `generations-unvendored` | ask | `step.generation` | `*` | provenance `Reviewed`, `Agent` | A |
| 15 | `downgrades` | ask | `downgrade` | `*` | | A |
| 16 | `boot-operations` | ask | `op.boot.*` | `*` | | S A |
| 17 | `base-instances` | ask | `step.*` | `@base` | | S |
| 18 | `base-changes` | ask | `step.write`, `step.delete`, `step.stop`, `step.restore-checkpoint`, `step.call` | `@base` | | A |
| 19 | `base-updates-unvendored` | ask | `step.update` | `@base` | provenance `Reviewed`, `Agent` | A |
| 20 | `retention` | ask | `step.*` | `/sys/config/store`, `/sys/config/journal` | | S A |
| 21 | `rule-of-two` | ask | `*` | `*` | rule of two; independent | S A |
| 22 | `budgets` | ask | `*` | `*` | budget above every kind | S A |

**`locked`** (L) lets agents only read, delegate and extract. **`supervised`** (S), the genesis preset, lets them apply steps up to `restorable` and asks for the rest. **`autonomous`** (A) also allows emissions to allowlisted destinations from untainted sessions, vendor updates and generations other than downgrades, and, with `agent_shared_space`, agent code in the shared space. In every preset `reviewers` holds `owner` and `human.**`, `sealers` these and `vendor.**`, `builders` `builder.**`, `budgets` one `agent.**` budget, and other fields their defaults.

## Policy proposals

`call policy propose(candidate)` checks a `Policy` by [validity](#validity), signature excepted, and creates an approval of kind `policy` whose facts are the rules it adds, removes and changes; each ask of the last 30 days it decides otherwise; and its rules matching no recorded subject or overridden by a stronger one. The owner accepts it by signing it, any session may write it, and the approval ends `allowed`. Proposals count toward the [ask limits](#expiry-and-limits).

## Approvals

An **approval** is the record of a plan's ask, at `/sys/approvals/a<n>` ([identifiers](02-identifiers.md#identifiers)). `policy` writes it when an apply or a `call` meets `Ask`, unless the plan hash has a pending or allowed one, which it reuses. `apply` stays `pending` while the approval is ([plan lifecycle](14-changes.md#plan-lifecycle)); a `call` fails with `policy.approval-required`, naming the approval and the plan hash to apply once allowed.

### Approval records

An approval is an `Approval` record holding its number; its kind, `plan` or `policy` for a proposal; the requester and session; the plan hash and read-set hash ([preview](14-changes.md#preview)), a proposal's candidate hash for both; its creation and expiry; the asking rules with their requirements; each subject's operation names, objects and condition values; the computed facts; the plan's `--intent`; its state and decisions. The link renders the facts first, as in this example, the legs of the rule of two following the taint when it holds, and each point of no return as its [outbox](14-changes.md#the-outbox) entry, a `Message` or `Spawn` delivery with its `code` and that bundle's [provenance class](09-authority.md#provenance). The intent follows, as untrusted content labelled `principal.<requester>` and the session's taint ([untrusted content](#untrusted-content)):

```text
approval a41, kind: plan, requester: agent.ops, session: s3, plan: afxanabvpwz4sxira4, expires: 2026-10-29T14:32:00.000Z~12ms
    rule no-return, decision: Decision.Ask, approvers: ["owner", "human.**"], quorum: 1, channel: Channel.Independent
    steps none: 1, recorded: 2, restorable: 0, irreversible: 1
    no-return step: 4, target: mail.send, destination: smtp.example.org:587, size: 3 KiB, blake3: ahvgou4wwjc76e5tto
    taint net.feeds.example.org: 14 min
    budget no-return: 3, limit: 10
    intent: envelope principal.agent.ops,net.feeds.example.org 1
    | send the weekly report to the finance list
```

### Decisions

A **decision** allows or denies an approval. Its principal is a `human.*` or `owner` principal that matches the approvers of an asking rule and is not the requester, an ancestor or a descendant of it (`policy.not-approver`). It arrives:

- by `approve` ([verbs](18-link.md#verbs)) over any session, carrying the principal's signature for the purpose `approval` ([signatures](09-authority.md#signatures)) over an `ApprovalDecision`;
- at a dedicated console, with presence and the principal's [PIN](#console-pins), without signature.

```nan
pub closed enum Constraint derives Encode {
    Once
    HashFor(seconds: u32)
} Constraint

pub closed struct ApprovalDecision derives Encode {
    pub approval: ApprovalId
    pub plan: HashValue
    pub read_set: HashValue
    pub allow: bool
    pub constraint: Constraint
    pub machine: Array[u8, 16]
} ApprovalDecision
```

- `approval`, `plan`, `read_set` and `machine` match the pending approval (`policy.decision-mismatch`, `policy.not-pending`) and the machine.
- `Once`, the default, covers one apply. `HashFor(seconds)` covers every apply of the same plan hash and read-set hash for at most 86,400 s; on a plan with a point of no return it fails with `policy.constraint-refused`.
- One deny by an approver denies the approval. It is allowed once, for every asking rule, `quorum` distinct principals among its approvers have allowed it.

### Channels

A rule whose `channel` is `Independent` accepts a decision only from a dedicated console ([`console`](20-base.md#console)), or over a session of the deciding principal on a connection carrying no session of an agent, of a child or of `inst.<moniker>`; another fails with `policy.channel`. In `dev` mode the console session counts as a dedicated console.

### Expiry and limits

- An approval pending for `expiry_min` minutes, the least among its asking rules', expires, and its plan fails with `policy.approval-expired`. An allowed `Once` approval not consumed within that time after its decision expires too, unless its plan is `scheduled`: it then stays allowed until the plan leaves `scheduled`, and the plan's apply consumes it ([timed plans](14-changes.md#timed-plans)).
- A family holds at most 10 pending approvals (`Policy.asks_pending`) and creates at most 30 per hour (`Policy.asks_per_hour`). An ask beyond fails with `policy.ask-limit` and writes no approval.

### Approval states

| State | Event | Next | Effect |
| --- | --- | --- | --- |
| none | an apply or `call` meets `Ask` | `pending` | `policy.asked`; the plan is `awaiting-approval` |
| `pending` | allows complete every asking rule's quorum | `allowed` | the waiting apply resumes, the plan is `scheduled` with `--when` ([timed plans](14-changes.md#timed-plans)), or the plan of a `call` is `previewed` again |
| `pending` | a deny | `denied` | the plan fails with `policy.approval-denied` |
| `pending` | `expiry_min` passes | `expired` | the plan fails with `policy.approval-expired` |
| `pending` | the plan's read set changes | `lapsed` | the plan fails with `plan.stale` |
| `pending` | `cancel` of the plan, or `freeze agents` | `withdrawn` | the plan fails with `plan.cancelled` |
| `allowed` | `Once` consumed, `HashFor` ends, or expiry | `expired` | the next apply asks again |

Leaving `pending` emits `policy.approval-ended`, and each decision `policy.decided`.

### Console PINs

At a dedicated console, a `human.*` or `owner` principal decides with its **PIN** of 6 to 12 decimal digits, which `keys` keeps outside history as an Argon2id tag (version 0x13, 8 MiB, 3 passes, parallelism 1, 16-byte random salt, 32-byte tag) and compares in constant time (requires [N25](22-nan.md#n25)).

- To set it, the human types `pin` at the console, which prints a 16-byte nonce; a session of the principal sends `call console pin(<nonce>)` within 120 s; the console then reads the PIN twice (`policy.pin-format`).
- A wrong PIN fails with `policy.pin-wrong` and emits `policy.pin-failed`. After 5 in a row the principal's PIN is locked for 10 min, across restarts (`policy.pin-locked`).
- Outside `dev` mode, PIN entry on a shared console fails with `policy.console-shared`.

## Taint

The **taint** of a session is a set of [origin labels](02-identifiers.md#origin-labels), each with the time it was received; `link` keeps each principal's labels across restarts and reboots ([`link`](20-base.md#link)).

### Origin sets

The **origin set** of a commit, kept in its metadata ([commits](11-store.md#commits)), is the union of:

- the taint of the session whose request caused it ([causes](17-observability.md#causes));
- the ingress labels of the instance that wrote it;
- `upload.<session>` for the blob of a `put`;
- the origin sets of the nodes a step copies from: the record a field write rewrites, the commits a revert restores, the sides of a merge.

An instance's **ingress labels** are those of the data its capabilities brought in since its spec was created: `net.<host>` of each flow of its `Egress` and `Listen` capabilities ([origin labels](16-network.md#origin-labels)), `device.<moniker>` of each `Console` or `Nic` driver that delivered it data, `model.<name>` of each model whose output `infer` returned it, and the labels that the sessions it opens through `LinkSession` ([transports](18-link.md#transports)) gain. They persist at `/state/<moniker>/labels` ([/state](13-tree.md#state)); `link` and `console` have none. An object's origin set is its last writing commit's; an event's is its cause's session taint with its source's ingress labels.

### Untrusted content

The link renders as **untrusted content**, in an envelope ([responses](18-link.md#responses)):

1. every `Untrusted` field;
2. every text, byte or blob value and node name whose author is neither `system`, nor a base instance without ingress labels, nor the reader's principal, an ancestor or a descendant of it;
3. every value and name of a node or event whose origin set is not empty.

A value's **author** is the principal of the commit, or the source of the event, recording it. Untrusted content is labelled with its origin set, and in case 2 with `principal.<author>`.

### Growth and sharing

A session's taint grows only:

1. when a response to it carries untrusted content, by the content's labels;
2. when a session of its principal or of an ancestor gains a label, by that label;
3. for a session of `inst.<moniker>`, when its instance gains an ingress label, by that label.

A session opens with the labels its principal's and its ancestors' sessions gained in the last 24 h and, for a session of `inst.<moniker>`, its instance's ingress labels, and its taint never shrinks while it is open. A quarantined session's labels reach no other session ([extraction](#extraction)). Growth emits `taint.grew`.

## Rule of two

A plan whose session's principal is of an agent's [family](#enrollment) **meets the rule of two** when it combines:

1. **untrusted input**: the session's taint is not empty, or its family received an untainted extraction result in the last hour;
2. **sensitive data**: a sensitive node's content was rendered to a session of its principal or of an ancestor in the last 24 h, or the plan's read set holds a sensitive node;
3. **an outward or irreversible effect**: a point of no return, or a `grant` step conferring an `Egress` capability.

A **sensitive node** is a `secret` record ([secret records](11-store.md#secret-records)) or a node whose path matches a glob of `Policy.sensitive`.

The presets' rule `rule-of-two` asks for each plan meeting it. The record of that ask ([errors and fixes](18-link.md#errors-and-fixes)) has the code `taint.rule-of-two`, names each leg with its labels, nodes and steps, and offers an approval (`fix 1 approval`) and, when an ancestor of the session's principal is untainted, a **clean child session**: the `delegate` request that ancestor sends for a child holding the plan's grants, in whose untainted session the plan is previewed again.

## Extraction

`extract <path> as <type> [with <extractor>]` ([verbs](18-link.md#verbs)) turns untrusted content into a typed value:

1. `link` opens a **quarantined session** of the requester's principal, without connection. Its only capabilities are a read-only `View` of the node at `<path>` and, for a model, a port to `infer` limited to `extract`.
2. The extractor runs there for at most 60 s, charged to the requester's account. By default, `link` decodes the node as `<type>`, a record of that type or a blob of its construction text, with the decoder that `<type>`'s declaring bundle exports ([canonical rendering](02-identifiers.md#canonical-rendering)) (requires [N20](22-nan.md#n20)). `with model.<name>`, `link` calls `infer`'s `extract` with that model through the session's port and decodes the construction text it returns. `with <hash>`, the admitted bundle whose typed entry takes `Untrusted[mem::Bytes]` and returns `<type>` runs as a [job](06-execution.md#jobs) ([job entries](06-execution.md#job-entries)) (requires [N29](22-nan.md#n29)); on a machine without `user-mode`, a job placed isolated fails the extraction with `place.unavailable` ([machine conditions](09-authority.md#machine-conditions)). Another type is refused (`taint.extractor-invalid`); a failure or an undecodable result fails with `taint.extract-failed`.
3. The result's **capacity** comes from the schema tree of `<type>` (requires [N20](22-nan.md#n20)): `bool` 1 bit, an integer its width, `char` 21, `f32` 32, `f64` 64, `Unit` and `Never` 0; a struct or layout the sum of its fields'; an enum, `Option` and `Result` included, ⌈log₂ Σ 2^c⌉ over its variants' payload capacities c. Predicates do not reduce it.
4. The result is **untainted** when `<type>` is a parsing type ([NaN 15 §Validator classes](https://github.com/nan-language/nan/blob/main/docs/spec/15-untrusted.md#validator-classes)) without `Untrusted` part and, for each label of the input's origin set and `model.<name>`, the bits the family extracted from it in the last hour plus the capacity stay within 64 (`Policy.extraction_bits`); those bits are then charged. Otherwise the result renders as untrusted content with those labels.
5. `taint.extracted` is emitted. An untainted result counts as untrusted input for the [rule of two](#rule-of-two) for one hour.

## Agent budgets

The first `AgentBudget` of `Policy.budgets` whose `principals` glob matches an enrolled agent principal limits its family. Daily counts restart at 00:00 UTC.

| Field | Counts | Default |
| --- | --- | --- |
| `no_return_per_hour` | points of no return applied in the last 60 min | 10 |
| `egress_bytes_per_day` | bytes the [egress gate](16-network.md#egress-gate) sent for the family's sessions | 64 MiB |
| `energy_mj_per_day` | energy charged to their sessions' [accounts](08-scheduling.md#accounting) | 36,000,000 mJ |
| `tokens_per_day` | inference tokens `infer` charged them | 1,000,000 |
| `money_per_day` | money `infer` charged them, in millionths of its configured currency | 0 |

A plan **exceeds** a budget when the count plus its previewed cost passes the limit, and the presets' rule `budgets` then asks. Activity outside plans counts, unstopped.

## Emergency controls

The **emergency controls** are `freeze agents`, `freeze egress` and `freeze all`, which `human.*` and `owner` principals invoke, and `rollback` and `recovery`, which only `owner` invokes. Each is invoked:

- at the console with presence, `rollback` and `recovery` on a dedicated console with the owner's PIN ([`console`](20-base.md#console));
- in a session of the principal: `freeze <scope>` ([oversight verbs](18-link.md#oversight-verbs)), `call boot rollback()` or `call boot recovery()`;
- on a connection without session, by `freeze <scope> --by <principal> --at <time>` with a signature for the purpose `request` ([signatures](09-authority.md#signatures)) over an `OversightRequest`.

```nan
pub closed enum Action derives Encode {
    FreezeAgents
    FreezeEgress
    FreezeAll
    Rollback
    Recovery
} Action

pub closed struct OversightRequest derives Encode {
    pub action: Action
    pub principal: Principal
    pub time_ns: i64
    pub machine: Array[u8, 16]
} OversightRequest
```

A signed request is accepted when its signature verifies (`auth.bad-signature`), its principal may invoke its action (`policy.not-human`), `machine` is the machine id (`policy.machine`), `time_ns` lies within 5 min of the wall time beyond the clock's uncertainty, and its hash is new (`policy.request-stale`). `link` serves emergency controls before the connection's queued requests and outside every budget; no rule, budget or approval applies to them.

| Control | Effect, in one commit by the invoking principal |
| --- | --- |
| `freeze agents` | removes every grant the principals of agents' families hold ([revocation](09-authority.md#revocation)): their sessions end, their approvals are `withdrawn`, and their plans awaiting approval or scheduled and their jobs are cancelled, a job with reason `freeze` ([branches and results](06-execution.md#branches-and-results)) |
| `freeze egress` | removes every grant of an `Egress` object; the egress gate resets their flows |
| `freeze all` | both, and sets `frozen` in `/sys/config/supervisor`, pausing the intake of every instance but the base instances ([instance specifications](06-execution.md#instance-specifications)) |
| `rollback` | `boot` boots the previous good generation ([generation operations](03-boot.md#generation-operations)) |
| `recovery` | `boot` sets `next once` to recovery and reboots ([recovery](03-boot.md#recovery)) |

A freeze ends by the `undo` of its commit ([undo](14-changes.md#undo)) in a session of a `human.*` or `owner` principal. Each control emits `policy.frozen` and enters the [audit chain](17-observability.md#audit-chain).

## Costs

Only pending approvals cost while idle, an expiry timer each. A decision costs one pass over the decoded rules per subject; an ask or a decision one commit and one signature check; a response with untrusted content one label-set union; an extraction one decode, plus one `infer` call or one job when `with` names its extractor.

## Events

| Kind | Payload | Section |
| --- | --- | --- |
| `policy.owner-mismatch` | both keys' fingerprints | [Owner](#owner) |
| `policy.claimed`, `policy.recovered` | key fingerprint, transport | [Claim](#claim) |
| `policy.changed`, `policy.invalid` | version, commit; code | [Validity](#validity) |
| `policy.notified` | plan, step, rule | [Evaluation](#evaluation) |
| `policy.asked`, `policy.approval-ended` | approval, plan, requester; approval, state | [Approval states](#approval-states) |
| `policy.decided` | approval, principal, allow, channel | [Decisions](#decisions) |
| `policy.pin-failed` | principal, failures in a row | [Console PINs](#console-pins) |
| `policy.frozen` | control, principal, commit | [Emergency controls](#emergency-controls) |
| `taint.grew` | session, labels | [Growth and sharing](#growth-and-sharing) |
| `taint.extracted` | session, labels, capacity, untainted | [Extraction](#extraction) |

## Codes

| Code | Meaning | Section |
| --- | --- | --- |
| `policy.machine` | a changed or foreign machine id | [Owner](#owner) |
| `policy.claim-code` | wrong or expired claim code | [Claim](#claim) |
| `policy.already-claimed` | the machine has an owner | [Claim](#claim) |
| `policy.unclaimable` | the loader can store no owner key | [Claim](#claim) |
| `policy.claim-declined` | no presence for the claim | [Claim](#claim) |
| `policy.recovery-code-invalid` | a recovery code matching no digest | [Recovery](#recovery) |
| `policy.agent-enroll` | `enroll` conferred on an agent | [Enrollment](#enrollment) |
| `policy.denied` | a subject's decision is `Deny` | [Evaluation](#evaluation) |
| `policy.version` | the version does not exceed the accepted one | [Validity](#validity) |
| `policy.pattern` | a pattern does not parse | [Validity](#validity) |
| `policy.agent-approver` | an approver, reviewer or sealer glob can match an agent | [Validity](#validity) |
| `policy.preset-unavailable` | `autonomous` or `agent_shared_space` without `iommu` | [Validity](#validity) |
| `policy.approval-required` | a `call` needs an approval | [Approvals](#approvals) |
| `policy.not-approver` | the principal may not decide it | [Decisions](#decisions) |
| `policy.decision-mismatch` | the decision names another approval, plan or machine | [Decisions](#decisions) |
| `policy.not-pending` | the approval is not pending | [Decisions](#decisions) |
| `policy.constraint-refused` | `HashFor` on a point of no return, or beyond 86,400 s | [Decisions](#decisions) |
| `policy.channel` | a decision over a refused channel | [Channels](#channels) |
| `policy.approval-expired` | the approval expired | [Expiry and limits](#expiry-and-limits) |
| `policy.ask-limit` | the family's ask limit is reached | [Expiry and limits](#expiry-and-limits) |
| `policy.approval-denied` | an approver denied | [Approval states](#approval-states) |
| `policy.pin-format` | the PIN is not 6 to 12 digits | [Console PINs](#console-pins) |
| `policy.pin-wrong` | wrong PIN | [Console PINs](#console-pins) |
| `policy.pin-locked` | the PIN is locked | [Console PINs](#console-pins) |
| `policy.console-shared` | PIN entry on a shared console | [Console PINs](#console-pins) |
| `policy.not-human` | the principal may not invoke the control | [Emergency controls](#emergency-controls) |
| `policy.request-stale` | the signed request is late or replayed | [Emergency controls](#emergency-controls) |
| `taint.rule-of-two` | the plan meets the rule of two | [Rule of two](#rule-of-two) |
| `taint.extractor-invalid` | the extractor does not produce the type | [Extraction](#extraction) |
| `taint.extract-failed` | the extractor failed or its result does not decode | [Extraction](#extraction) |
