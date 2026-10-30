# The link

This chapter defines the link protocol: its transports and framing, the ready record, connections and sessions, requests and their operands, responses, pushed records, the twenty verbs, queries and selectors, errors and fixes, budgets, the primer, wake rules, the sandbox serial binding, the link's configuration, and the codes of the `request`, `link` and `budget` families.

## Transports

The **link** is the [base instance](20-base.md#base-instances) `link`, serving one protocol on every transport. A **transport** carries **connections**, each a two-way stream of lines.

| Transport | Endpoint | Connections | A session opens with |
| --- | --- | --- | --- |
| serial | the console UART while the console is shared ([console](20-base.md#console)) | one, from the link's start to the machine's halt or power off | a challenge and a signature |
| vsock | virtio-vsock port 7710 | one per stream | a challenge and a signature |
| TCP | port 7710 under Noise_IK ([network transports](16-network.md#network-transports-of-the-link)) | one per stream | the Noise handshake |
| port | `LinkSession.open` | one per `open` | the opening instance's [principal](09-authority.md#principals) `inst.<moniker>` |

The port transport serves resident agents, instances granted a `Port[LinkSession]`:

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `LinkSession.open` | `client: Port[LinkClient]` | a connection | `link.session-limit` | `none` |
| `LinkSession.send` | `line: text::String`, one request line | — | — | `irreversible{emit,actuate,erase,spend}` |
| `LinkClient.block` | `text: text::String`, one block | — | — | `none` |

## Framing

- A connection carries UTF-8 lines ending in LF, holding no other control character. A request holds at most 1 MiB (`request.too-large`).
- Bytes before a connection's ready record, such as firmware output on serial, are no link output.
- When a connection opens, and on serial when the link starts listening, the link writes the **ready record**:

```text
ok 0 link: records 1
link version: 1, mode: "normal", generation: 7, claimed: true, cursor: e1042, key: ahta22o7mkjx4jvr62
ok 0 link: records 1
```

```nan
pub closed struct LinkReady derives Encode {
    pub version: u32
    pub mode: text::String
    pub generation: u64
    pub claimed: bool
    /// The cursor of the `events` stream.
    pub cursor: Cursor
    /// The hash of the link's Ed25519 identity public key.
    pub key: HashValue
} LinkReady
```

- `link`'s published summary ([/live](13-tree.md#live)) is a `LinkSummary`, `ready_ns` being the monotonic time of the boot's first ready record.

```nan
pub closed struct LinkSummary derives Encode {
    pub ready_ns: u64
    pub sessions: u32
} LinkSummary
```

- The link writes each **block**, a response or a pushed block, whole: the lines of two blocks never interleave.
- On serial, NaN's kernel halt report ([NaN 24 §Kernel halts](https://github.com/nan-language/nan/blob/main/docs/spec/24-kernel.md#kernel-halts)) may appear between blocks; its lines start with `trap.` or `halt `, and no line of the link starts so.
- When the console shares the link's UART, console blocks carry the id `console` and precede pending link output, and input lines starting `console ` go to the console ([console](20-base.md#console)).

## Connections and sessions

A **session** is a [principal](09-authority.md#principals)'s authenticated presence on a connection: a root capability minted from the principal's [grants](09-authority.md#grants) ([sessions as authority](09-authority.md#sessions-as-authority)), a `SessionBudget` ([budgets](#budgets)) and a [taint](19-oversight.md#taint). Sessions are named `s<n>`. On serial and vsock a session opens in three steps:

1. `hello <principal>` answers a `challenge` record: a 32-byte nonce in checksummed base32, the connection's binding in full form, and the challenge's expiry, 60 s after it is issued (`LinkConfig.challenge_s`).
2. The client signs a `HelloProof` for the purpose `handshake` ([signatures](09-authority.md#signatures)), the object being its `mem::wire` encoding, and sends `hello <principal> <signature>`.
3. The link verifies the signature against the principal's enrolled key and opens the session. An unknown principal fails with `link.principal-unknown`, a wrong signature with `link.signature-invalid`, an expired or replaced challenge with `link.challenge-expired`.

```nan
pub closed struct HelloProof derives Encode {
    pub principal: Principal
    pub nonce: Array[u8, 32]
    pub binding: HashValue
} HelloProof
```

- The **binding** of a serial or vsock connection is the hash of kind `handshake` ([hashes](02-identifiers.md#hashes)) of the text `<transport> <boot counter> <connection number> <full form of LinkReady.key>`, and of a TCP connection the Noise handshake hash. On TCP, `hello <principal>` opens the session when the handshake's static key is the principal's.
- In `dev` mode, `hello owner` on serial opens without a signature ([modes](03-boot.md#modes)).
- An unclaimed machine outside `dev` mode serves only `status`, `primer` and `hello --claim <code> <public key> [--encrypt]` ([claim](19-oversight.md#claim)); another request fails with `link.unclaimed`. Before a session opens, a connection serves only `hello`, `status`, `primer` and a signed `freeze` (`link.no-session`).
- A child principal opens its sessions over any connection, signing with its credential ([delegation](09-authority.md#delegation)).
- A connection carries at most 16 sessions (`LinkConfig.sessions`, `link.session-limit`). With one open, requests use it; with several, each request names one with `--session s<n>` (`link.session-required`). A request id is unique among the connection's pending requests and watches (`request.duplicate-id`). A client must give each mutating request, unless it retries an earlier one, an id that no earlier mutating request of its session used ([request keys](14-changes.md#request-keys)).
- A detached session keeps its taint, watches, cursors, pending requests and kept results. `hello --resume s<n>` answers a challenge as `hello` does. A `--resume` or `--session` naming no session of the principal fails with `link.session-unknown`.
- A principal holds at most 16 sessions, connected or detached (`LinkConfig.principal_sessions`); opening another fails with `link.session-limit`.
- A restart of `link` ends every session, detached ones included, with its watches and pending requests, whose final blocks never come; applying [plans](14-changes.md#plans-and-steps) fail with `plan.interrupted` ([procedure](14-changes.md#procedure)). `link` then writes the ready record again on serial, and every vsock, TCP and port connection closes. A `hello --resume` naming a session the restart ended fails with `link.session-unknown`, and an `apply` sent again with its request key returns its plan's state when a span commit carries that key ([request keys](14-changes.md#request-keys)).

| State | Event | Next | Effect |
| --- | --- | --- | --- |
| — | it opens on a connection ([transports](#transports)) | `connected` | `link.session-opened` |
| `connected` | its connection is lost | `detached` | its blocks queue; `link.session-detached` |
| `detached` | `hello --resume s<n> <signature>`, or on TCP `hello --resume s<n>` when the handshake's static key is the principal's | `connected` | the current connection receives the queued blocks; `link.session-resumed` |
| `detached` | 10 min pass (`LinkConfig.detach_s`) | `ended` | `link.session-ended` |
| `connected`, `detached` | the grant it opened under expires or is revoked, by a `freeze` among others ([sessions as authority](09-authority.md#sessions-as-authority), [emergency controls](19-oversight.md#emergency-controls)), or its principal's key is revoked ([rotation and revocation](09-authority.md#rotation-and-revocation)) | `ended` | `link.session-ended` |

A request naming an ended session fails with `link.session-ended`, and its applying [plans](14-changes.md#plans-and-steps) continue.

`/live/sessions/<id>` holds each session's `LiveSession` ([/live](13-tree.md#live)):

```nan
pub closed struct LiveSession derives Encode {
    pub principal: Principal
    pub transport: Transport
    /// `link`, its device's driver and, for vsock and TCP, `net`.
    pub path: mem::List[Moniker]
    pub connected: bool
    pub opened: WallTime
    /// The end of its reattach window while detached.
    pub expires: Option[WallTime]
    pub taint: mem::List[OriginLabel]
    pub budgets: mem::List[BudgetUse]
    pub watches: u32
    /// The session whose `delegate` created its principal.
    pub parent: Option[SessionId]
    pub as_of: u64
} LiveSession

pub closed struct BudgetUse derives Encode {
    pub name: text::String
    pub used: u64
    pub limit: Option[u64]
} BudgetUse

pub closed enum Transport derives Copy, Eq, Encode {
    Serial
    Vsock
    Tcp
    Port
} Transport
```

## Requests

A request is one logical line, `<id> <verb> <operands> [--flags]`. It continues over lines while a delimiter is open or a line ends with `,`, as NaN source does; an empty line while a delimiter is open ends the request, which fails with `request.unbalanced`.

```ebnf
request = REQUEST_ID VERB operand* flag* ;
operand = ATOM | TEXT | form ;
form = ATOM? "(" items? ")" | "[" items? "]" | "{" JSON_MEMBERS? "}" ;
items = item ( "," item )* ","? ;
item = ( LABEL ":" )? operand+ flag* ;
flag = FLAG operand? ;
```

Tokens are separated by single spaces outside forms and text literals. `REQUEST_ID` is a request id ([identifiers](02-identifiers.md#identifiers)); `VERB` a verb or `status`; `ATOM` a run of characters other than spaces, `,`, `"` and `()[]{}`; `TEXT` a NaN text literal; `LABEL` a NaN field or parameter name followed by `: ` inside a form; `JSON_MEMBERS` the members of a JSON object; `FLAG` `--` and a lower-case word with hyphens. Each verb types its operands:

| Operand | Form |
| --- | --- |
| path | `/` and segments ([paths](02-identifiers.md#paths)), a segment outside `[A-Za-z0-9._@+=-]` written as a text literal |
| moniker, principal | as [monikers](02-identifiers.md#monikers) and [principals](02-identifiers.md#principals) define them |
| [commit](11-store.md#transactions) | `#<n>`, or a [branch](11-store.md#branches) name standing for its head |
| hash, [plan](14-changes.md#plans-and-steps) | a short or full hash form ([hashes](02-identifiers.md#hashes)) |
| cursor, [cause](17-observability.md#causes), approval, session | `e<n>`, `c<boot>.<n>`, `a<n>`, `s<n>` ([ids](02-identifiers.md#identifiers)) |
| fault | `f<n>` ([fault records](17-observability.md#the-journal-and-its-streams)) |
| value | a NaN construction literal decoded against the operand's type (requires [N19](22-nan.md#n19), [N20](22-nan.md#n20)), or with `--json-in` a JSON value of it ([NaN 13 §JSON](https://github.com/nan-language/nan/blob/main/docs/spec/13-text-encode.md#json)) |
| steps, call | a step list and an operation call ([steps](14-changes.md#steps)) |
| time | RFC 3339 UTC, or `+<n>s` from now |

A line the grammar rejects fails with `request.malformed`, an operand of the wrong form with `request.invalid-operand`, a name naming nothing with `request.unknown-name`, a name or short hash matching several with `request.ambiguous`, an unknown verb with `request.unknown-verb` and an unknown flag with `request.invalid-flag`. The flags of every verb:

| Flag | Effect |
| --- | --- |
| `--session s<n>` | the session the request runs in |
| `--key <key>` | the request key of a mutating request ([request keys](14-changes.md#request-keys)) |
| `--budget <units>` | this block's output budget, at least 100 units |
| `--skip <n>` | leaves out the first n records |
| `--fields <names>` | prints only these fields; `hash` prints full hashes |
| `--json`, `--json-in` | JSON Lines output; JSON values as operands |

`status` is `get /live/status`.

## Responses

A response is a block: a verdict line, records, then the same verdict line.

- The **verdict line** is `<word> <id> <verb> <first operand>: records <n>`, n counting every record printed or elided, followed by `, units <u>`, the block's size, when the session has a session output budget. The word is `ok`, `failed` or `pending`. The first operand prints in canonical form, cut to 64 bytes ending in `...`; a step list prints as `[<n> steps]`.
- A `failed` block holds at least one `error` record ([errors and fixes](#errors-and-fixes)).
- A request that has not ended within 100 ms (`LinkConfig.pending_ms`) answers `pending`, with a `pending` record giving its state; its final block, with the same id, follows as a pushed block when it ends.
- A **record** is a header line at column 0, then continuation lines indented 4 spaces. The header holds its kind word, its subject when its kind has one, then its other fields in declaration order as `name: value`, separated by `, ` and rendered canonically ([canonical rendering](02-identifiers.md#canonical-rendering)). A field that is `None` or an empty list is left out, and `Some(v)` prints as `v`. This chapter writes a record as its kind, `<subject>` and its field names.
- Continuation lines hold the `- ` and `+ ` lines of old and new values, the lines of errors, `node` values and envelopes.
- An **envelope** prints each piece of [untrusted content](19-oversight.md#untrusted-content) as the continuation line `<field>: envelope <origins> <n>` followed by n lines `| <text>`, `<origins>` being its [origin labels](02-identifiers.md#origin-labels) separated by `,`; its header field is left out (requires [N29](22-nan.md#n29)). Envelopes are written by a declassifying validator of `link` sealed by a `vendor.*` [principal](09-authority.md#principals) (requires [N15](22-nan.md#n15)).
- A block ends with a `taint` record naming the origin labels the session's [taint](19-oversight.md#taint) gained since its previous block, if any: `taint added: [net.mail.example.org]`.
- A block holds at most the response budget ([budgets](#budgets)), counted in units ([units](02-identifiers.md#units)): the most records after the skipped ones, at least one, such that the verdict lines, those records and the elision line fit. The **elision** line `elided <n> records: <verb> <operands> --skip <k>` is the request, without its id, that prints the rest.
- With `--json`, each line is a JSON object, members in byte order of their names, without whitespace outside strings: a verdict line `{"id":I,"operand":O,"record":"verdict","summary":{"records":N},"verb":V,"verdict":W}`; a record `{"record":K,"value":V}`, V in `text::json`'s encoding, an envelope `{"envelope":ORIGIN,"lines":[...]}` standing for its field's value; an elision `{"count":N,"record":"elided","request":R}`.

## Pushed records

A **pushed block** is a block written without a request of its own: the final block of a pending request, or a watch's events. A watch's block is `ok <watch id> watch: records <n>` and holds `event`, `commit` and `lost` records in cursor order. The link writes a watch's block within 100 ms (`LinkConfig.push_ms`) of its first undelivered match, within the response budget, and coalesces repeats of one kind from one source into one record with `count: <n>`. A watch holding more than 1,024 undelivered matches (`LinkConfig.backlog`) drops the oldest and reports them as `lost count: <n>, since: <cursor>`; `watch --since <cursor>` reads them again while their stream retains them.

```text
ok 5 watch: records 2
event e4410, time_ns: 3127000000, source: app.feeds, kind: "instance.failed", severity: Severity.Error, cause: c12.1, principal: system, payload: InstanceFailed(code: "trap.exhausted", restarts: 5, escalation: Some(app))
lost count: 12, since: e4380
ok 5 watch: records 2
```

## Verbs

| Verb | Operands | Records |
| --- | --- | --- |
| `hello` | `<principal> [<signature>]`; `--claim <code> <public key> [--encrypt]`; `--resume s<n> [<signature>]` | `challenge`; `session`, `system` |
| `primer` | `[<topic>]` | `primer` ([the primer](#the-primer)) |
| `delegate` | a `Delegation` ([delegation](09-authority.md#delegation)) | `principal`, `credential` |
| `describe` | `<path>`, `<moniker>`, `<protocol>`, `<type>` or `<code>` | `api`; `code` |
| `get` | `<path> [--at <commit>]` | `node`; `dir`, `entry`; `blob`, `piece`; `stream` |
| `find` | `<query>` | the rows, printed as `get` prints them; `event`, `row` |
| `diff` | `<a> <b> [<path>]` | `added`, `removed`, `changed` |
| `history` | `<path>`, `<moniker>` or `<commit>`; `--replay <seq> [--bundle <hash>] [--with-head]` | `commit`; `input`; `replay`, `changed`, `output` |
| `why` | `<path>`, `<commit>`, `<event>`, `<fault>` or `<cause>` | `because` |
| `watch` | `<selector> [--since <cursor>] [--wake]` | `watch`; pushed `event`, `commit`, `lost` |
| `call` | `<moniker> <op>(<args>)` and [plan](14-changes.md#plans-and-steps) options | `reply`, then as `apply` |
| `plan` | `<steps>` and plan options ([options](14-changes.md#options)) | the effect set ([the effect set](14-changes.md#the-effect-set)) |
| `apply` | `<plan> [--when <time>]` | `pending`; `span`, `delivery`, `plan` |
| `undo` | `<commit>` or `<plan>` | the effect set, `irreversible` |
| `confirm` | `<commit>` | `confirmed` |
| `cancel` | `<request id>`, `<plan>`, `<moniker>` or `<principal>` | `cancelled` |
| `put` | `<path> <size> <blake3>` and data lines | `blob` |
| `extract` | `<path> as <type> [with <extractor>]` | `extracted` |
| `approve` | `<approval> allow` or `deny`, `[<constraint>] <signature>` | `approval` |
| `freeze` | `agents`, `egress` or `all`; without a session, also `<signature> --by <principal> --at <time>` | `frozen` |

Failures are those of [authority](09-authority.md#grants) and of each verb's linked section.

### Session verbs

- `hello` opens a session ([connections and sessions](#connections-and-sessions)) and prints `session <id>, principal, transport, path, mode, generation, expires, taint` and `system cpus, memory_bytes, instances, converged, approvals, cursor`; `hello --claim` is the claim.
- `delegate` creates a child principal ([delegation](09-authority.md#delegation)) and prints `principal <name>, expires` and `credential <name>, seed`, the seed being the child's 32-byte private key seed in checksummed base32, printed once.

```text
> 1 hello agent.ops
ok 1 hello agent.ops: records 1
challenge nonce: mftsgs7y5fvbjpvqlyy42c6vhtz44gsdtxoimc6xvofjfyzerz3qub, binding: aeico6y6hr2pvqomolccyavzym57pgegiotb3e7ilbuncwf2pvnxkep, expires: 2026-10-29T12:00:41.000Z~12ms
ok 1 hello agent.ops: records 1
> 2 hello agent.ops 2rplegz5uor74eeye3dnnj6mz2ks65vkhgcq6iv624s4f6mygqllkpfqicp3qyvtso77nxvkz5homeagsv2gqqsf7cg7gk7njdcpudakq
ok 2 hello agent.ops: records 2
session s3, principal: agent.ops, transport: Transport.Serial, path: [link, devices.uart0], mode: "normal", generation: 7
system cpus: 4, memory_bytes: 536870912, instances: 14, converged: 14, approvals: 0, cursor: e1042
ok 2 hello agent.ops: records 2
```

### Reading verbs

- `describe` prints an item as `nan show --api` does, `api <name>, kind, hash` with its declaration, contracts and digest as continuation lines; a node's type and writers; an [instance](06-execution.md#instances)'s spec, protocol and status. `describe <code>` prints `code <code>, section`, the meaning and an example from the code catalogue the generation ships.
- `get` prints a record as `node <path>, type, commit, hash` with its value on a continuation line; a dir as `dir <path>, entries` and an `entry <name>, kind, type, commit` per child; a blob as `blob <path>, size_bytes, blake3, origin` and a `piece <offset>, size_bytes` per 4 KiB, enveloped, as text when UTF-8 and else as base64; a stream as `stream <path>, count, first, last`; a `/live` node as `node` without [commit](11-store.md#transactions) ([/live](13-tree.md#live)). `--at` reads at a commit or [branch](11-store.md#branches).
- `find` runs a query ([queries](#queries-and-selectors)) and prints each node it finds as `get` prints it, each event as an `event` record ([pushed records](#pushed-records)), and each other record of a stream as `row <cursor>, time` with its value on a continuation line, a fault's subject being its id `f<n>`.
- `diff` compares `<a>` and `<b>`, each a commit or a branch, under `<path>` when given: `added` and `removed` records with `+` or `-` lines and `changed` records with the differing fields, in path order.
- `history` of a path prints a `commit #<n>, time, principal, session, plan, cause` record per commit writing under it, newest first, with its intent in an envelope; of a [moniker](02-identifiers.md#monikers), an `input <seq>, cause, sender, operation, size, hash` record per input header of its flight recorder ([flight recorders](17-observability.md#flight-recorders)); of a commit, its record and `changed` records. `--replay <seq>`, with `--bundle` and `--with-head`, replays that input ([replay](17-observability.md#replay)) and prints its report as `replay <seq>, bundle, result`, `changed` and `output` records.
- `why` prints the chain from its subject back to a request as `because <role>, subject, time` records, roles `written`, `sent`, `read`, `trapped`, `requested`, `planned`, `approved` and `scheduled` ([causes](17-observability.md#causes)).
- `watch` answers `watch <cursor>` and pushes the selector's matches ([pushed blocks](#pushed-records)) until `cancel <watch id>` or the end of its session.

```text
> 3 get /sys/config/app.feeds
ok 3 get /sys/config/app.feeds: records 1
node /sys/config/app.feeds, type: "feeds::FeedsConfig", commit: #5121, hash: aftwqo4cb62dimyo6i
    FeedsConfig(batch: 0, refresh_s: 300)
ok 3 get /sys/config/app.feeds: records 1
> 4 find from /live/accounts select cpu_ns order cpu_ns desc limit 5 --budget 100
ok 4 find from /live/accounts select cpu_ns order cpu_ns desc limit 5: records 5
node /live/accounts/app.indexer, type: "Account"
    Account(cpu_ns: 41000000000)
elided 4 records: find from /live/accounts select cpu_ns order cpu_ns desc limit 5 --budget 100 --skip 1
ok 4 find from /live/accounts select cpu_ns order cpu_ns desc limit 5: records 5
```

### Acting verbs

- `plan`, `apply`, `undo` and `confirm` preview, apply, invert and confirm plans as [changes](14-changes.md#plans-and-steps) defines. `apply` prints, while pending, `pending <state>, span, commit, until, approval`, the last naming the approval it awaits, and at its end one `span <n>, commit` record per span, one `delivery <entry>, status` record per outbox entry, and `plan <hash>, state`. `undo` adds an `irreversible <entry>, target, class` record per effect no undo reaches. `confirm` prints `confirmed #<n>, plan`.
- `call` is the one-step plan `plan [call <moniker> <op>(<args>)]` applied at once, answering as `apply` does with a `reply` record before the others. When the plan needs an approval, `call` fails with `policy.approval-required` ([approvals](19-oversight.md#approvals)) and an `approval <id>, plan, expires` record; `apply <plan>` applies it once approved.
- `cancel` classifies its operand: a [principal](09-authority.md#principals) starting with `inst.`, `agent.` or `human.`; else a hash form, a plan; else the id of a pending request or watch of the session; else, holding a `.` or naming an [instance](06-execution.md#instances), a moniker; else a request id of the session. It cancels a pending request or a watch, a plan awaiting approval or scheduled ([plan lifecycle](14-changes.md#plan-lifecycle)), a job with acknowledgement ([jobs](06-execution.md#jobs)), or the delegation of a child of the session's principal, ending the child's sessions. It prints `cancelled <target>`; a target that has committed or ended fails with `request.not-cancellable`.

```text
> 7 apply age7pwuq3zqxfkwosw
pending 7 apply age7pwuq3zqxfkwosw: records 1
pending PlanState.HealthWindow, span: 1, commit: #5131, until: 2026-10-29T12:02:31.000Z~12ms
pending 7 apply age7pwuq3zqxfkwosw: records 1
ok 7 apply age7pwuq3zqxfkwosw: records 3
span 1, commit: #5131
span 2, commit: #5132
plan age7pwuq3zqxfkwosw, state: PlanState.Applied
ok 7 apply age7pwuq3zqxfkwosw: records 3
```

### Data verbs

- `put <path> <size> <blake3>` uploads a blob: lines of base64 (RFC 4648 §4) follow the request line until `<size>` bytes are decoded. At most 64 MiB pass in one `put` (`request.too-large`); bytes whose blob id differs from `<blake3>` fail with `request.hash-mismatch`. The blob commits at the path with the origin `upload.<session>`, and `put` prints `blob <path>, size_bytes, blake3, commit, origin`.
- `extract` runs an extraction ([extraction](19-oversight.md#extraction)) and prints `extracted <path>, type, tainted` with its value, in an envelope when tainted.

```text
> 9 put /data/agent.ops/pkgs/hello.txt 6 aghey7a3thn72uhhok
> aGVsbG8K
ok 9 put /data/agent.ops/pkgs/hello.txt: records 1
blob /data/agent.ops/pkgs/hello.txt, size_bytes: 6, blake3: aghey7a3thn72uhhok, commit: #5140, origin: upload.s12.3
ok 9 put /data/agent.ops/pkgs/hello.txt: records 1
```

### Oversight verbs

- `approve` carries an approval decision, its constraint a `Constraint` value, `Once` by default, and its signature, over any session ([approvals](19-oversight.md#approvals)), and prints `approval <id>, allow, constraint, plan, principal`.
- `freeze` applies an emergency control ([emergency controls](19-oversight.md#emergency-controls)). Without a session, `--by` and `--at` give the principal and the time of the `OversightRequest` that the signature, for the purpose `request`, covers. It prints `frozen <scope>, sessions, grants, plans, jobs`.

```text
> 10 approve a40 allow xmslvwxvnate3t3zox5itwk6n6esblt5kwtv5klbszt34qqhld4mxt6lx54drg22fuk4lbtbsqtp2jyuofscwwzogh4fqamnzya2muyfj
ok 10 approve a40: records 1
approval a40, allow: true, constraint: Constraint.Once, plan: afgcntrrtpngsmhwpd, principal: owner
ok 10 approve a40: records 1
```

## Queries and selectors

`find` and `watch` take the rest of their line, flags aside, as a query and a selector.

```ebnf
query = 'from' PATH ( 'where' condition )? ( 'select' field ( "," field )* )? ( 'order' field ( 'asc' | 'desc' )? )? ( 'limit' NUMBER )? ;
selector = PATH ( 'kind' KIND ( "," KIND )* )? ( 'severity' ">=" SEVERITY )? ( 'where' condition )? ;
condition = term ( ( 'and' term )+ | ( 'or' term )+ )? ;
term = comparison | "(" condition ")" ;
comparison = field COMPARATOR LITERAL ;
field = NAME ( "." NAME )* ;
```

`COMPARATOR` is `==`, `!=`, `<`, `<=`, `>`, `>=`, `prefix` or `contains`; `LITERAL` a NaN literal, a variant name, a hash form, a [commit](11-store.md#transactions), a cursor or an RFC 3339 time; `KIND` an event kind, or a family followed by `.*`; `SEVERITY` one of `debug`, `info`, `notice`, `warn`, `error`, `critical`. `and` and `or` mix only through parentheses. `prefix` tests a text or path prefix; `contains` a list element or a substring.

- A query's rows are a dir's children, a stream's records, the commits of `/history/commits` or a `/live` subtree's nodes; `name`, `path` and `commit` name a node's name, path and last writing commit. `order` sorts ascending by default; `limit` keeps the first n rows.
- A query's cost bound, computed before it runs, is the records it scans, a dir's subtree count, a stream's count within the `seq` and `time` bounds of its top-level `and` terms or a `/live` subtree's node count, and the bytes it reads, those records times their mean size. A bound beyond the session's query budget fails with `budget.query`, whose fixes narrow the query to a sub-path or to a stream's most recent records within the budget. Dir children are indexed by name, stream records by `seq` and `time`; `order` on an indexed field with a `limit` ends the scan at the limit.
- A selector's path selects: under `/journal/<stream>`, that stream's new records; under `/live/instances/<moniker>`, the events whose source is that [moniker](02-identifiers.md#monikers) or one below it; under `/sys` or `/data`, the commits writing under the path, as `commit` records listing the paths written; `/history/commits`, every commit. `kind` and `severity` filter events, and `where` compares scalar fields of the event, its payload (`payload.restarts`) or the commit, never texts.
- A watch starts after `--since <cursor>`, a cursor `e<n>` of events or a commit `#<n>`, or else at the current end.

## Errors and fixes

An `error` record reports one root cause:

- the header `error <code> <subject>`, the subject being the path, [moniker](02-identifiers.md#monikers) or operand at fault, or `-`;
- `expected: ` and `found: ` lines;
- `because <role> <subject>` lines, roles as `why` gives them;
- `fix <n> <request>` lines, each a request without its id that the link has dry-run: parsed, authorized and, for a [plan](14-changes.md#plans-and-steps), previewed without this failure;
- `fix <n> approval <role>` followed by `+ ` lines holding the record a [principal](09-authority.md#principals) of that role signs, for a fix only a human can make.

An unknown name offers at most 3 fixes, the request with each nearest candidate by edit distance. `describe <code>` explains any code.

```text
> 11 apply age7pwuq3zqxfkwosw
failed 11 apply age7pwuq3zqxfkwosw: records 1
error plan.stale /sys/config/app.feeds
    expected: commit #5121
    found: commit #5133
    because written #5133 plan ah4cehzwkv3piud4sp
    fix 1 plan [write /sys/config/app.feeds.batch 32]
failed 11 apply age7pwuq3zqxfkwosw: records 1
```

## Budgets

A session's budget is the `budget` of its principal's `Session` grant, or `LinkConfig.budget` under a `Machine` grant, which all the principal's sessions consume together ([sessions as authority](09-authority.md#sessions-as-authority)); a field left `None` is unlimited.

```nan
pub closed struct SessionBudget derives Encode {
    pub output_units: Option[u64]
    pub response_units: u32
    pub query_records: u64
    pub query_bytes: u64
    pub previews: u32
    pub cpu_ms: Option[u64]
    pub memory_bytes: Option[u64]
    pub egress_bytes: Option[u64]
    pub energy_mj: Option[u64]
    pub infer_tokens: Option[u64]
    /// Millionths of the currency `infer` charges in.
    pub infer_micros: Option[u64]
} SessionBudget
```

| Budget | Fields | Default | When spent |
| --- | --- | --- | --- |
| output per block | `response_units` | 4,000 units | the block elides |
| total output | `output_units` | unlimited | requests but `hello` fail with `budget.attention`, uncounted; watches lose their matches |
| query | `query_records`, `query_bytes` | 100,000 records, 64 MiB | `budget.query` |
| previews | `previews` | [preview](14-changes.md#preview) | `budget.previews` |
| CPU, memory, egress, energy, inference | `cpu_ms`, `memory_bytes`, `egress_bytes`, `energy_mj`, `infer_tokens`, `infer_micros` | unlimited | `budget.exhausted`, naming the budget |

The session's [account](08-scheduling.md#accounting) carries its CPU, I/O and energy; a spent budget's fix is the delegator's [plan](14-changes.md#plans-and-steps) raising it.

## The primer

The **primer** is the text `primer` prints: the core, at most 3,000 units, and with `primer <topic>` one topic, at most 1,500 units each. Each prints as a `primer <part>` record whose continuation lines are the text. The primer is part of the generation's [kernel image](04-kernel.md#kernel-image).

| Part | Contents |
| --- | --- |
| core | requests; verdicts, records, envelopes, `taint` records, elisions and budgets; pushed records and wake rules; the [plan](14-changes.md#plans-and-steps) lifecycle; request keys; [taint](19-oversight.md#taint) and the rule of two; resumption; modes; the verbs; the topics |
| `tree` | the top-level nodes and their writers, node kinds, [commits](11-store.md#transactions) and `--expect`, `/live`, resolution |
| `query` | the query and selector grammars, indexes, cost bounds, cursors |
| `plans` | steps and options, the effect set's records, spans, the outbox, health windows, provisional commits, undo, timed plans |
| `code` | bundles, `put`, `install`, admission, placement, updates, probation, jobs, generations |
| `devices` | `/live/devices`, bindings in `/sys/config/devices`, device protocols, DMA rules |
| `net` | network configuration, `Egress` and `Listen` [grants](09-authority.md#grants), the egress gate, link transports |
| `oversight` | policy, presets, approvals, taint, the rule of two, extraction, emergency controls, the claim |
| `errors` | the error record, fixes, candidates, `describe <code>`, the code families and their chapters |

## Wake rules

A **wake rule** is a watch made with `--wake`. A session holding a wake rule receives pushed blocks only when a wake rule matches, or when a watch's undelivered matches reach `LinkConfig.backlog`; the block then holds the undelivered matches of all its watches. Final blocks of pending requests are pushed at once. On the port transport a block is a `LinkClient.block` message, which revives a hibernated resident agent ([lifecycle](06-execution.md#lifecycle)).

## The sandbox serial binding

In the sandbox ([the sandbox](01-machines.md#the-sandbox)), the serial transport is the machine's serial port, which `tools/sandbox start` binds to its standard input and output (requires [N22](22-nan.md#n22)). The development loader boots the machine in `dev` mode. Standard input carries request lines; standard output, firmware and loader lines, then the ready record and blocks. A client writes a request line and reads up to the second verdict line with its id. The console shares the port.

## Configuration

`/sys/config/link` holds the link's `LinkConfig`, whose defaults the rules above state; `budget` serves sessions under a `Machine` [grant](09-authority.md#grants), and `plans` configures [changes](14-changes.md#options).

```nan
pub closed struct LinkConfig derives Encode {
    pub budget: SessionBudget
    pub pending_ms: u32
    pub challenge_s: u32
    pub detach_s: u32
    pub sessions: u32
    pub principal_sessions: u32
    pub push_ms: u32
    pub backlog: u32
    pub plans: PlanConfig
} LinkConfig

pub closed struct PlanConfig derives Encode {
    pub health_s: u32
    pub confirm_s: u32
    pub preview_bytes: u32
    pub keep_s: u32
    pub activation_ms: u32
    pub grace_s: u32
    pub key_results: u32
} PlanConfig
```

## Events

| Kind | Payload | Section |
| --- | --- | --- |
| `link.session-opened`, `link.session-detached`, `link.session-resumed`, `link.session-ended` | session, principal, transport | [Sessions](#connections-and-sessions) |
| `link.hello-rejected` | principal, transport, failure code | [Sessions](#connections-and-sessions) |

## Codes

| Code | Meaning | Section |
| --- | --- | --- |
| `request.ambiguous` | a name or short hash matches several | [Requests](#requests) |
| `request.duplicate-id` | the id is pending on the connection | [Sessions](#connections-and-sessions) |
| `request.hash-mismatch` | the upload's blob id differs | [Data verbs](#data-verbs) |
| `request.invalid-flag` | an unknown flag, or a flag value of the wrong form | [Requests](#requests) |
| `request.invalid-operand` | an operand of the wrong form or value | [Requests](#requests) |
| `request.key-reused` | a request key names a different request | [Request keys](14-changes.md#request-keys) |
| `request.malformed` | the line is not a request | [Requests](#requests) |
| `request.not-cancellable` | the target has committed or ended | [Acting verbs](#acting-verbs) |
| `request.too-large` | a request or upload beyond its limit | [Framing](#framing), [Data verbs](#data-verbs) |
| `request.unbalanced` | an empty line ended an open request | [Requests](#requests) |
| `request.unknown-name` | a name names nothing | [Requests](#requests), [Errors and fixes](#errors-and-fixes) |
| `request.unknown-verb` | an unknown verb | [Requests](#requests) |
| `link.challenge-expired` | the signature answers no live challenge | [Sessions](#connections-and-sessions) |
| `link.no-session` | the request needs a session | [Sessions](#connections-and-sessions) |
| `link.principal-unknown` | the principal has no enrolled key | [Sessions](#connections-and-sessions) |
| `link.session-ended` | the session ended | [Sessions](#connections-and-sessions) |
| `link.session-limit` | the connection or the principal holds its maximum of sessions | [Sessions](#connections-and-sessions) |
| `link.session-required` | `--session` is missing among several sessions | [Sessions](#connections-and-sessions) |
| `link.session-unknown` | no session of the principal has that id | [Sessions](#connections-and-sessions) |
| `link.signature-invalid` | the `hello` signature does not verify | [Sessions](#connections-and-sessions) |
| `link.unclaimed` | the machine is unclaimed | [Sessions](#connections-and-sessions) |
| `budget.attention` | the session's output budget is spent | [Budgets](#budgets) |
| `budget.exhausted` | another session budget is spent | [Budgets](#budgets) |
| `budget.memory-pressure` | memory pressure refuses an instance start or a limit increase | [Pressure](05-memory.md#pressure) |
| `budget.over-reservation` | reserved memory exceeds RAM less the core reserve, or a CPU reservation fits on no CPU | [Reserved instances](05-memory.md#reserved-instances), [Shares and reservations](08-scheduling.md#shares-and-reservations) |
| `budget.previews` | the principal's sessions run their maximum of previews | [Preview](14-changes.md#preview) |
| `budget.query` | a query's cost bound exceeds the query budget | [Queries and selectors](#queries-and-selectors) |
