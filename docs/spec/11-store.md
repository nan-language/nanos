# The store

This chapter defines the store: nodes and their kinds, declared types and validity, identity, versions, transactions, commits, history and retention, branches, merges and conflicts, revert and diff, streams, unversioned nodes, secret records, file views for `sys::Fs`, the `Store` protocol, and the codes of the `store`, `schema` and `conflict` families.

## Nodes

The **store** is NaN OS's one versioned, transactional, content-addressed store, served by the base instance `store` ([20-base.md](20-base.md#store)) from one volume ([12-store-format.md](12-store-format.md#placement)). It holds a tree of nodes per [branch](#branches), one machine tree for `/journal`, and [commit](#commits) metadata. A node sits at a [path](02-identifiers.md#paths) and has one of four kinds:

| Kind | Holds |
| --- | --- |
| **record** | one NaN `Encode` value as canonical `mem::wire` bytes ([NaN 13 §Wire format](https://github.com/nan-language/nan/blob/main/docs/spec/13-text-encode.md#wire-format)), with its type |
| **blob** | bytes, stored in 64 KiB pieces |
| **dir** | children by name, and a declaration |
| **stream** | append-only typed entries with sequence numbers and times |

A dir lists its children in byte order of their names, a numbered dir in numeric order. Its declaration is `DirDecl { element: Option[Element], ephemeral: bool, secret: bool, numbered: bool, keep: Option[Keep] }`: `Element { kind: NodeKind, type: Option[TypeRef] }` gives every child's kind, and the type of record and stream children; an `ephemeral` dir's descendants are [unversioned](#unversioned-nodes); every record created below a `secret` dir is a [secret record](#secret-records); the store names each child added with `Add` to a `numbered` dir by the next decimal number, from 1; `Keep { count: u32, age_s: Option[u64] }` gives the children a numbered dir of unversioned nodes keeps.

`ephemeral`, `secret` and `numbered` never change once the dir exists (`schema.mismatch`); `keep` applies only to numbered dirs of unversioned nodes (`store.kind`). A stream's declaration is `StreamDecl { type: TypeRef, max_age_s: Option[u64], max_bytes: Option[u64] }`; a `Declaration` is `Dir(DirDecl)` or `Stream(StreamDecl)`.

## Declared types and validity

A `TypeRef { name: text::String, schema: HashValue }` names a NaN type by its canonical path ([NaN 29 §Canonical paths](https://github.com/nan-language/nan/blob/main/docs/spec/29-identity-versioning.md#canonical-paths)) and its schema hash ([NaN 13 §Schema hashes](https://github.com/nan-language/nan/blob/main/docs/spec/13-text-encode.md#schema-hashes)). The **declared type** of a record node or stream is its dir's element type, else the type of the write that creates it. A record write must carry the declared type's name and a schema hash equal to the declared one, or whose tree accepts or is accepted by the declared one ([NaN 29 §Encode class](https://github.com/nan-language/nan/blob/main/docs/spec/29-identity-versioning.md#encode-class)); otherwise it fails with `schema.mismatch`. When the written tree accepts the declared one, the declared type takes the written schema. The bytes must decode as the type:

- A **typed write** comes from an instance through a [`View`](09-authority.md#capability-kinds) whose write operation encodes the value with its type's derived encoder, so predicates and invariants hold by construction (requires [N20](22-nan.md#n20), [N27](22-nan.md#n27)).
- Any other write is an **encoded write**: bytes from the link, an upload, a merge or a message. The store decodes it with the decoder exported by the admitted bundle or kernel image declaring the type, predicates and invariants included, and fails it with `schema.invalid`, naming the `core::DecodeError` and field path, when the decoder rejects it (requires [N20](22-nan.md#n20)).

The **schema registry** maps a schema hash to its tree, type name and decoder, from the kernel image and the admission records under `/sys/code` ([10-code.md](10-code.md#admission)), whose code roots carry a schema tree per `Encode` type (requires [N20](22-nan.md#n20)); an unknown schema hash fails with `schema.unknown`. A child whose kind or type differs from its dir's element fails with `schema.mismatch`. Stream entries validate as records of the stream's type. Blobs have no type; their bytes read back `Untrusted` ([NaN 15 §Untrusted values](https://github.com/nan-language/nan/blob/main/docs/spec/15-untrusted.md#untrusted-values)).

## Identity

Hashes are NaN hash values ([NaN 29 §Hash values](https://github.com/nan-language/nan/blob/main/docs/spec/29-identity-versioning.md#hash-values)) of the bytes named:

| Node | Hash |
| --- | --- |
| record | `record`, LF, the 33 bytes of its schema hash and its wire bytes |
| secret record | `secret`, LF and its stored bytes ([12-store-format.md](12-store-format.md#secret-records)) |
| blob | none: its **blob id** is the tag `0x01` and the BLAKE3-256 digest of its bytes |
| stream, ephemeral dir | `stream` or `ephemeral`, LF and the wire bytes of its declaration |
| other dir | `dir`, LF, the wire bytes of its declaration and its entry digest |

A dir's **entry digest** covers its children in name order. Each child gives an entry hash, of `entry`, LF, its name as wire text, its kind as one byte (1 record, 2 blob, 3 dir, 4 stream) and its hash. A dir without children has the hash of `run` and LF as digest. Otherwise its entry hashes are cut into runs after each hash whose last byte is below 16 and after the last one; each run gives the hash of `run`, LF and its hashes; run hashes are cut the same way, level by level, while more than one remains, and the one left is the digest.

The **root hash** of a commit is the hash of a dir holding only its `data` and `sys`. The store computes it for branch heads at each [index flush](12-store-format.md#index-flushes) and on request only.

## Versions

Each commit takes the next **commit number** `#<n>` ([02-identifiers.md](02-identifiers.md#identifiers)) of one counter per store, shared by all branches; genesis is #1. A node's **version** is the number of the commit that last wrote it, as its branch sees it; a dir's changes with its declaration and its set of children. A dir's **subtree version** is the number of the last commit that changed it or a descendant. Operations name a version as `Version`: `Commit(CommitId)`, `Hash(HashValue)` for a node's hash, or `Absent`.

## Transactions

A transaction reads a snapshot, its base, and commits writes atomically to one branch; the store takes no locks. Its record is `Transaction { base: Option[CommitId], reads: mem::List[Read], expect: mem::List[Expect], writes: mem::List[Write], key: Option[text::String], plan: Option[HashValue], intent: Option[text::String] }`: `base` is the commit its reads saw, `None` making it blind, based on the head at commit; a `Read { path, version: Version, scope }` has the scope `Node` or `Subtree`; an `Expect { path, version: Version }` is an explicit condition; `writes` apply in order; `key` is its [request key](14-changes.md#request-keys); `plan` is the plan hash when it applies a step of a [plan](14-changes.md#plans-and-steps).

| Write | Effect |
| --- | --- |
| `Put { path, value: Value }` | creates or replaces a record or blob; `Value` is `Record { type: TypeRef, bytes: mem::Bytes, secret: bool }` or `Blob(HashValue)`, a blob the store holds |
| `Make { path, declaration: Declaration }` | creates a dir or stream, or changes a dir's element |
| `Delete { path }` | removes a node and its descendants |
| `Add { path, value: Value }` | adds a child named by the next number to a numbered dir |
| `Append { path, entries: mem::List[mem::Bytes] }` | appends entries to a stream |
| `Expire { path, at: Option[WallTime] }` | sets or clears an unversioned node's expiry |
| `Hold { path, on: bool }` | holds or releases a child of a numbered dir with `keep` |
| `Trim { path, below: u64 }` | drops a stream's entries below sequence `below` |

`Put`, `Make` and `Add` create missing ancestors as dirs with an empty declaration. The store stamps a commit's principal, session or instance, cause, origin set and time from the [`View`](09-authority.md#capability-kinds) and the message header, never from the transaction.

### Isolation

- **Snapshot isolation.** A commit fails with `conflict.write` when a node it writes, or a node below a dir it deletes, has a version above its base. Adding or removing a child does not write its dir.
- **Serializable commits.** A transaction that writes below `/sys` or carries a plan hash also fails with `conflict.read` when one of its reads no longer holds at the head: a `Node` read compares the node's version, a `Subtree` read the dir's subtree version.

### Expectations and request keys

An `Expect` holds when the node at the head has the version or hash it names, or, for `Absent`, when no node exists; otherwise the commit fails with `conflict.expect`.

A transaction whose request key the store holds for the same principal writes nothing: it returns the commit made under that key when both have the same digest, the hash of `transaction`, LF and the wire encoding of the writes, expectations, plan and intent, and fails with [`request.key-reused`](18-link.md#codes) otherwise. The store keeps, across reboots, the keys that [request keys](14-changes.md#request-keys) keep.

### Limits

A transaction holds at most 10,000 writes, 64 MiB of record and stream entry bytes, and 65,536 reads and expectations. A record holds at most 4 MiB of wire bytes, a stream entry 64 KiB, an intent 1 KiB and a blob 2^40 bytes. Exceeding a limit fails with `store.too-large`. Blob bytes travel through drafts ([The Store protocol](#the-store-protocol)) and count against no transaction limit.

### Commit procedure

One sequencer per store takes transactions in arrival order:

1. Resolve each path through the `View`'s subtree, branch and rights: `store.denied`. In recovery mode, a versioned write that [recovery](03-boot.md#recovery) does not allow: `store.read-only`.
2. Look up the request key, as above.
3. Check the limits: `store.too-large`.
4. Check each write's kind and validity: `store.kind`, `schema.*`.
5. Validate the write set: `conflict.write`.
6. For a serializable commit, validate the reads: `conflict.read`.
7. Check the expectations: `conflict.expect`.
8. Check space ([Thinning](#thinning)): `store.full`.
9. Number the commit, append it to the log, and reply `Committed { number: CommitId, branch, time: WallTime }` when its log flush returns ([12-store-format.md](12-store-format.md#group-commit)); a device failure fails every commit of that log flush with `store.io` and leaves the store read-only; such a commit exists from the next mount on exactly when that mount replays it, and its request key then returns it.

A transaction that fails before step 9 has no effect and takes no number. A commit is **durable** when its log flush returns, and visible from then on in number order. Its cost is its validation, one log append, index updates in memory and at most two device flushes ([21-performance.md](21-performance.md)). An idle store does no work.

## Commits

A **commit** is the result of one transaction on a branch. Its metadata is `CommitMeta { number: CommitId, branch: text::String, parents: mem::List[CommitId], kind: CommitKind, principal: Principal, session: Option[SessionId], instance: Option[Moniker], cause: Cause, plan: Option[HashValue], intent: Option[text::String], origins: mem::List[OriginLabel], key: Option[text::String], time: WallTime, recorded: bool, root: Option[HashValue] }`:

- `parents` holds the previous head; for a merge, the target's head then the source's; none for #1.
- `kind` is `Write`, `Merge { source: text::String }`, `Revert { of: CommitId }` or `Genesis`.
- `principal` is the principal on whose authority the commit is made: the session's, or the one the writing instance [acts for](09-authority.md#principals). `session` or `instance` names the writer, the runner's writes for an instance naming that instance. `cause` ([17-observability.md](17-observability.md#causes)) says why.
- `origins` is its sorted origin set, `time` the wall time at numbering, `root` its root hash once computed.

For its [origin set](19-oversight.md#origin-sets), `link` supplies a session's taint with its View and the runner an instance's ingress labels. Commits are never modified.

## History and retention

A commit is **recorded** when it changes a versioned node: a node below `/sys` or `/data` outside every ephemeral dir. **History** is the retained recorded commits of every branch. A [View](09-authority.md#capability-kinds) reads its branch's versioned nodes as of any retained commit of the branch's ancestry; another commit fails with `store.thinned`. An unrecorded commit's metadata goes at the next index flush.

### Retention

`Retention { tiers: mem::List[Tier], max_pct: u8, agent_pct: u8 }` is the `retention` field of `/sys/config/store` ([13-tree.md](13-tree.md#sys)). Each `Tier { every_s: u64, age_s: u64 }` keeps, among commits younger than `age_s`, one per `every_s` seconds, or all when `every_s` is 0. By default `tiers` keeps every commit younger than 1 h, one per 10 min younger than 24 h, one per hour younger than 7 days and one per day younger than 90 days; `max_pct` and `agent_pct`, the percentages of capacity that history and [agent bytes](#thinning) may hold, are 25 each, from 0 to 50.

A recorded commit is retained when it is a branch head, when a pin holds it, or when the tier with the smallest `age_s` above its age keeps every commit or names an interval of `every_s` seconds whose latest flush commit ([12-store-format.md](12-store-format.md#index-flushes)) it is. A `Retention` write that thins a retained commit has the reversibility `irreversible{erase}` ([14-changes.md](14-changes.md#reversibility)).

### Pins

A **pin** holds a commit against thinning: `Pin { commit: CommitId, reason, by: Principal, note: Option[text::String] }`, reason `Generation`, `Approval`, `Plan`, `Branch` or `Explicit`. `boot` pins the commit at which each generation in the boot record was installed ([03-boot.md](03-boot.md#boot-record)); a pending [approval](19-oversight.md#approvals) pins its plan's base, and a running plan its own ([14-changes.md](14-changes.md#preview)); a branch pins its fork commit; principals pin with `pin`. The system removes its pins when their reason ends.

### Thinning

Thinning removes whole commits, with the node versions no retained commit shares, at index flushes and when a write of a principal other than an agent or a child principal needs space, never on a timer. The next retained commit on the branch takes over a thinned commit's changes for diff, revert and history.

**Agent bytes** are the bytes added on the authority of [agents and child principals](09-authority.md#principals): the node versions and stream entries their commits write, while a branch head or a retained commit reaches them, and their drafts that no node references. **History bytes** are the live bytes, agent bytes excepted, that only retained commits other than branch heads reach. When history bytes exceed `max_pct` of the capacity, or a write of a principal other than an agent or a child principal needs space the volume lacks, the store thins unpinned commits other than heads, from the oldest, and emits `store.history-thinned`. An agent's or child principal's commit, append or draft append that raises agent bytes above `agent_pct` of the capacity, or needs space the volume lacks, fails with `store.full` and thins nothing. When only pins and heads remain and space still lacks, every commit adding data fails with `store.full` and the store emits `store.writes-refused`; deletes, unpins, erasures and branch deletions still commit.

## Branches

A **branch** is a movable name for a commit, created in O(1) by sharing every node of its fork commit. `main` is the branch the machine runs; the **running branch** is `main`, or `trial/<n>` while generation n runs on it ([health gate](03-boot.md#health-gate)). A [View](09-authority.md#capability-kinds) names one branch, by default the running branch.

```ebnf
BRANCH = BRANCH_SEGMENT ( "/" BRANCH_SEGMENT )* ;
BRANCH_SEGMENT = ( "a" .. "z" | "0" .. "9" ) ( "a" .. "z" | "0" .. "9" | "." | "_" | "-" )* ;
```

A branch name is at most 64 bytes. `main` and the prefixes `trial/`, `job/` and `preview/` belong to the system; other names are session branches. A branch's record is `Branch { name, head: CommitId, fork: CommitId, parent: Option[text::String], created_by: Principal, expires: Option[WallTime] }`. A branch has its own unversioned nodes, shared with its parent's at creation. `branch` returns a View of the same subtree and rights on the new branch. A taken name fails with `store.branch-exists`; a reserved or malformed name, or deleting `main` or the running branch, with `store.branch-reserved`.

### Trial branches

1. When generation n boots in mode `trial` after a [generation switch](10-code.md#generation-switch), the store creates `trial/<n>` at the head of `main` unless one forked at or after generation n's install commit (`Generation.base` in [/sys/boot](13-tree.md#sysboot)) exists; creating it replaces one forked before that commit.
2. Generation n runs on `trial/<n>`; `main` does not change.
3. When the [health gate](03-boot.md#health-gate) passes, `boot` has the store take `trial/<n>` in one step of the sequencer: it fast-forwards `main` to `trial/<n>`, or merges `trial/<n>` into `main` when `main` moved and gives `main` its unversioned nodes, makes `main` the running branch, and deletes `trial/<n>`; every [View](09-authority.md#capability-kinds) naming `trial/<n>` names `main` from then on. A conflict keeps `trial/<n>` running and emits `store.trial-merge-conflict`; an `owner` merge of `trial/<n>` into `main` with `take` completes the take.
4. When the loader gives the trial slot up, `main` runs again as it was, and `trial/<n>` stays for inspection until step 1 replaces it or a [retirement](03-boot.md#generation-operations) raising the floor above n deletes it.

### Job and preview branches

A [job](06-execution.md#jobs) writes to `job/<moniker>`. At its successful end, the store merges that branch into the job's target branch as one commit, even where a fast-forward is possible, and deletes it; a conflict fails the job with `conflict.merge`. A kept job branch expires after its job's `keep_h` hours ([06-execution.md](06-execution.md#branches-and-results)), and creating `job/<moniker>` replaces it. A [preview](14-changes.md#preview) writes to `preview/<n>`, n a number the store assigns.

## Merges and conflicts

`merge` brings a source branch's changes into a target branch, through a [View](09-authority.md#capability-kinds) on the target. Its base is the latest commit both share: the fork commit of the one created from the other, or the source head of their last merge. A node inside the View changed on one side since the base takes that side's value, and one changed on both sides to the same value keeps it; a record changed on both sides, with one type, merges field by field; any other pair of changes conflicts. A field merge compares the wire records key by key, recursing into records and maps and taking other values whole; a field changed on both sides to different values conflicts. The merged bytes are an encoded write, so a value its decoder rejects is a conflict. Dirs merge child by child; declarations, blobs and streams merge whole. Unversioned nodes are not merged: a fast-forward takes the source's with its head, a merge keeps the target's.

A merge with conflicts fails with `conflict.merge`, commits nothing, and returns one `Conflict { path, field: Option[text::String], kind, base: Option[HashValue], source: Option[HashValue], target: Option[HashValue] }` per conflict, kind `Both`, `DeleteChange`, `KindChange`, `TypeChange` or `Invalid`, each with a [fix](18-link.md#errors-and-fixes) taking one side.

`take` lists `Take { path, side: Side.Source or Side.Target }`, resolving each listed conflict by taking that side's node whole. When the target's head is the base, the merge fast-forwards the target and makes no commit.

## Revert and diff

`revert` writes a commit of kind `Revert` restoring each versioned node a commit changed, inside the [View](09-authority.md#capability-kinds), to its value in the commit's parent on its branch. When a commit after it on the branch changed the same node or field, compared as in a merge, it fails with `conflict.revert` and one `Conflict` per node, writing nothing. Reverting a commit no longer retained fails with `store.thinned`, naming the retained commit holding its changes.

`diff` compares a subtree at two points, each a retained commit or a branch head, and returns `Edit { path, change, from: Option[HashValue], to: Option[HashValue], fields: mem::List[FieldEdit] }` in path order: `change` is `Added`, `Removed` or `Changed`, and each `FieldEdit { field: text::String, from: Option[mem::Bytes], to: Option[mem::Bytes] }` holds a differing record field's wire bytes. It skips every subtree both trees share, so its cost grows with the changes, not the tree.

## Streams

A stream holds entries `StreamEntry { seq: u64, time: WallTime, value: mem::Bytes }`, numbered consecutively from 1, each holding the wire bytes of a value of the stream's type. The store drops entries older than `max_age_s` or beyond `max_bytes`, from the oldest, at appends and index flushes, never on a timer. `read_stream` returns entries from a sequence number on, with `lost` counting requested entries already dropped.

## Unversioned nodes

**Unversioned nodes** are the nodes below `/state` ([13-tree.md](13-tree.md#state)), the descendants of an **ephemeral dir**, and the entries of streams. They keep their present value only, survive reboots, and stay out of history, diff, revert, root hashes and every reversibility guarantee: writing one has the reversibility `none` ([14-changes.md](14-changes.md#reversibility)). Reading one at a commit other than its branch's head fails with `store.unversioned`.

A numbered dir with `keep` deletes its lowest-numbered child at each `Add` beyond `count`, and its children older than `age_s`; a held child is neither counted nor deleted. An unversioned node or a branch may carry an **expiry**. A node past its age or expiry is invisible from that moment and removed at the next index flush.

## Secret records

A **secret record** is stored encrypted under a key of its own, its **node key**, which only `keys` derives ([12-store-format.md](12-store-format.md#secret-records)): the log, the index and history hold only its envelope. A record is secret when the `Put` creating it says so or a dir above it is declared `secret`; it stays secret for its life, and its versions on all branches share its key. A blob below a `secret` dir fails with `store.kind`.

The store validates a secret write like any other and has it encrypted through its `KeyUse(node)` ([09-authority.md](09-authority.md#keys-on-the-machine)); it never holds a node key and never decrypts. A [View](09-authority.md#capability-kinds) never yields a secret record's plaintext to an instance in the shared space: it gives its size and schema hash, and the link shows only that. The plaintext is reached only through operations of `keys` on a `KeyUse(node)`, or by an instance in an isolated space whose View carries the right `secret-read`, its read having `keys` open the envelope (requires [N23](22-nan.md#n23)). `erase` destroys the key, then deletes the node from the View's branch in one commit; from then on every version of the node, on any branch, in history or in an exported copy, fails to read with `store.erased`.

## File views

On NaN OS, a `sys::Fs` ([NaN 14 §Std capabilities](https://github.com/nan-language/nan/blob/main/docs/spec/14-authority.md#std-capabilities)) is a [View](09-authority.md#capability-kinds) of a dir, whose paths are relative to that dir, and a file is a blob node (requires [N1](22-nan.md#n1), [N27](22-nan.md#n27)).

`open(path)` gives a `File` reading the blob at the path as the View sees it at the call, and `create(path)` one writing a new draft, creating missing dirs; `File.write` appends to the draft, and the release of a writing `File` writes the draft as the path's blob. `read_all`, `write_all`, `remove` and `byte_len` read a blob, write one, delete a node and give a size.

A path whose segments break the [path](02-identifiers.md#paths) rules fails with `FsError.InvalidPath`. Naming a record, dir or stream where a file is expected, or a write the View's rights or the node's kind or type refuse, fails with `FsError.Denied`; `store.not-found` gives `NotFound`, `store.full` gives `NoSpace`, a revoked View `Revoked`.

For an instance, each `write_all`, `remove` and release of a writing `File` is one blind commit on the View's branch; `write_all` and `remove` return once it is durable. A job's writes go to its branch, and its end commits them as one ([Job and preview branches](#job-and-preview-branches)). A release cannot report a failure: one whose commit fails emits `store.file-lost` and loses the file's bytes, unless it failed with `store.io` and the next mount replays it ([commit procedure](#commit-procedure)); one after the View's revocation discards the draft.

## The Store protocol

`store` serves the `Store` protocol (requires [N4](22-nan.md#n4)). Each operation arrives through a [`View`](09-authority.md#capability-kinds), whose subtree, branch, rights and as-of apply to every path. Reads need `read` rights; `commit`, `append`, drafts, `revert` and `erase` need `write`; `branch`, `delete_branch`, `merge`, `pin`, `unpin`, `scrub` and `gc` need `admin`, and `keyring_seed` a View covering `/state/keys`, which only `keys` holds; a missing right fails with `store.denied`, and a revoked View fails as [revocation](09-authority.md#revocation) states.

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `get` | path, at: `Option[CommitId]` | `Node` | `store.not-found`, `store.thinned`, `store.unversioned`, `store.erased`, `store.corrupt` | `none` |
| `list` | path, at, after: `Option[text::String]`, limit: `u32` | `Listing` | as `get`, `store.kind` | `none` |
| `count` | path, at, from, to: `Option[text::String]` | `Count { children: u64, bytes: u64 }` | as `list` | `none` |
| `scan` | path, at, from, to, limit | `mem::List[Node]` with values | as `list` | `none` |
| `read` | path, at, offset: `u64`, length: `u32`, verify: `bool` | blob bytes, `Untrusted` | as `get`, `store.kind` | `none` |
| `map` | path, at, offset, length | a read-only mapping ([05-memory.md](05-memory.md#mapped-blobs)) | as `read` | `none` |
| `draft` | — | a draft id | `store.full` | `none` |
| `draft_append` | draft, bytes | — | `store.not-found`, `store.full`, `store.too-large` | `none` |
| `draft_close` | draft, expect: `Option[HashValue]` | blob id and size | `store.not-found`, `store.hash-mismatch` | `none` |
| `commit` | `Transaction` | `Committed` | those of the [commit procedure](#commit-procedure) | `recorded`; `none` for unversioned writes |
| `append` | path, entries | the sequence number of the earliest new entry | `store.kind`, `store.too-large`, `store.full`, `schema.*` | `none` |
| `read_stream` | path, from: `u64`, limit | entries and `lost` | `store.not-found`, `store.kind` | `none` |
| `subscribe` | prefix, since: `CommitId`, port | a subscription id | `store.thinned` | `none` |
| `unsubscribe` | subscription id | — | `store.not-found` | `none` |
| `history` | path, before: `Option[CommitId]`, limit | `mem::List[CommitMeta]`, newest to oldest | `store.not-found` | `none` |
| `diff` | from, to: each a commit or a branch, path | `mem::List[Edit]` | `store.thinned`, `store.not-found` | `none` |
| `branch` | name, at: `Option[CommitId]` | its `Branch` record and a View of it | `store.branch-exists`, `store.branch-reserved`, `store.thinned` | `recorded` |
| `delete_branch` | name | — | `store.not-found`, `store.branch-reserved` | `irreversible{erase}` |
| `merge` | source: a View of the source branch, take | `Committed` | `conflict.merge`, `store.denied` | `recorded` |
| `revert` | commit, dry_run: `bool` | `Committed`, or the writes when `dry_run` | `conflict.revert`, `store.thinned` | `recorded` |
| `pin` | commit, note | — | `store.thinned` | `restorable` |
| `unpin` | commit | — | `store.not-found` | `irreversible{erase}` |
| `erase` | path | `Committed` | `store.kind`, `store.not-found` | `irreversible{erase}` |
| `scrub` | — | `ScrubReport` ([12-store-format.md](12-store-format.md#scrub)) | `store.io` | `none` |
| `gc` | — | commits thinned and bytes freed | — | `none` |
| `stats` | — | `LiveStore` ([13-tree.md](13-tree.md#live)) | — | `none` |
| `keyring_seed` | secret id | its seed | `store.denied`, `store.erased` | `none` |

`Node` is `{ path, kind, version: CommitId, subtree: Option[CommitId], declared: Option[TypeRef], hash: Option[HashValue], size: u64, origins: mem::List[OriginLabel], secret: bool, expires: Option[WallTime], declaration: Option[Declaration], value: Option[Untrusted[mem::Bytes]] }` (requires [N29](22-nan.md#n29)); `size` counts bytes, children or entries. `Listing` is `{ entries: mem::List[Entry], next: Option[text::String] }`, `Entry` holding a child's name, kind, version, size and type name. `count` reads the index's subtree counts ([12-store-format.md](12-store-format.md#index-trees)).

A draft is a blob in the making that no node references: `draft_append` adds bytes, `draft_close` computes the blob id, failing with `store.hash-mismatch` when it differs from `expect`, and a `Put` or `Add` references it. A draft unreferenced 1 h after its last append is dropped, and no draft survives a reboot. A `read` with `verify` checks the blob id over the bytes it returns; the link and the [egress gate](16-network.md#egress-gate) set it for bytes leaving the machine.

`subscribe` sends one-way messages to the [port](07-communication.md#ports) it is given, for commits after `since`: `Changed { subscription, commit: CommitId, paths: mem::List[Path], more: u64 }` per commit changing a node below the prefix, with at most 64 paths and `more` counting the rest; `Appended { subscription, path, seq: u64 }` per stream append; and, once the subscriber's [credit](07-communication.md#credits) is spent, one `Lost { subscription, commits: u64 }` for what it could not send. What the store reports to a View names other commits by number only, and only paths inside the View.

## Events

| Kind | Payload | Section |
| --- | --- | --- |
| `store.history-thinned` | `{ commits: u64, bytes: u64, oldest: CommitId }` | [Thinning](#thinning) |
| `store.writes-refused` | `{ free: u64, history: u64 }` | [Thinning](#thinning) |
| `store.trial-merge-conflict` | `{ branch, conflicts: u32 }` | [Trial branches](#trial-branches) |
| `store.file-lost` | `{ path, code }` | [File views](#file-views) |

## Codes

| Code | Meaning | Section |
| --- | --- | --- |
| `store.not-found` | nothing of that name in the View | [The Store protocol](#the-store-protocol) |
| `store.denied` | the path is outside the View, or the View lacks the right | [Commit procedure](#commit-procedure) |
| `store.kind` | the operation or declaration does not apply to the node's kind | [Nodes](#nodes) |
| `store.too-large` | a limit is exceeded | [Limits](#limits) |
| `store.full` | no space remains after thinning every unpinned commit but heads, or an agent's or child principal's write lacks space or exceeds `agent_pct` | [Thinning](#thinning) |
| `store.thinned` | the commit is not retained | [History and retention](#history-and-retention) |
| `store.unversioned` | a read at a past commit reaches an unversioned node | [Unversioned nodes](#unversioned-nodes) |
| `store.erased` | the secret record's key was destroyed | [Secret records](#secret-records) |
| `store.hash-mismatch` | a blob's id differs from the expected one | [The Store protocol](#the-store-protocol) |
| `store.read-only` | recovery mode or a failed device refuses writes | [Commit procedure](#commit-procedure) |
| `store.branch-exists` | a branch of that name exists | [Branches](#branches) |
| `store.branch-reserved` | the name is malformed or reserved, or the branch is undeletable | [Branches](#branches) |
| `store.corrupt` | a checksum, magic, decoding or identity check failed | [12-store-format.md](12-store-format.md#conventions) |
| `store.io` | the device failed a read or a write | [12-store-format.md](12-store-format.md#group-commit) |
| `store.no-volume` | no partition or file can hold the store | [12-store-format.md](12-store-format.md#placement) |
| `store.format-unsupported` | the volume's format is neither the kernel's nor the previous one | [12-store-format.md](12-store-format.md#format-versions-and-upgrades) |
| `store.locked` | the volume key cannot be unsealed | [12-store-format.md](12-store-format.md#volumes) |
| `conflict.write` | a written node changed after the transaction's base | [Isolation](#isolation) |
| `conflict.read` | a read of a serializable commit no longer holds | [Isolation](#isolation) |
| `conflict.expect` | an expectation does not hold | [Expectations and request keys](#expectations-and-request-keys) |
| `conflict.merge` | a merge found a node or field changed on both sides | [Merges and conflicts](#merges-and-conflicts) |
| `conflict.revert` | a subsequent commit changed what a revert restores | [Revert and diff](#revert-and-diff) |
| `schema.invalid` | the declared type's decoder rejects the bytes | [Declared types and validity](#declared-types-and-validity) |
| `schema.mismatch` | the type or kind is not the declared one, or a fixed field changes | [Declared types and validity](#declared-types-and-validity) |
| `schema.unknown` | the registry knows no type of that schema hash | [Declared types and validity](#declared-types-and-validity) |
