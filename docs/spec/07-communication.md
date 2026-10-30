# Communication

This chapter defines protocols and their hashes, ports and their attenuation, wiring and its compatibility check, sends, calls and replies with their failures and pending replies, credits, ordering, headers, the transfer of messages within a bundle and between bundles and spaces, capability transfer, and the channels of the isolated transport.

## Protocols

A **protocol** is a NaN `protocol` item: a named set of operations, each with its head, contracts, digest and declared reversibility bound ([reversibility](14-changes.md#reversibility)) (requires [N4](22-nan.md#n4)). An operation without a result is **one-way**; an operation with a result type `R` and a failure type `E` is a **request**. A protocol has at most 256 operations, numbered in declaration order; admission refuses a larger one (`port.too-many-operations`).

- The **message enum** of a protocol has one variant per operation, holding its arguments. A request's variant also holds the request's `nanos::ReplyId[R, E]`, and each capability argument arrives in it as a `nanos::CapId[C]` ([runner tables](06-execution.md#runner-tables)).
- An instance **serves** the protocol whose message enum is its component's `M`. Only an instance that serves a protocol receives messages from other instances.
- An operation declared without `pub` is **internal**: only the serving instance sends it, through `Cx.send`, `Cx.after` and its keyed tasks' outboxes, and no port's mask holds it.
- A protocol may declare the summary type that its servers publish ([held effects](06-execution.md#held-effects)).
- Arguments, results and failures implement `Encode`. A field that carries data from outside the sender's principal must have type `Untrusted[T]` (requires [N29](22-nan.md#n29)). Protocol types are the same in every placement.

The **protocol hash** covers each operation's name, kind and head, the schema hashes of its types and its reversibility bound (requires [N4](22-nan.md#n4)). The **operation hash** covers the same facts for one operation.

## Ports

A **port** is a `nanos::Port[P]`: a Copy capability that sends messages of protocol `P` to one instance (requires [N27](22-nan.md#n27)). It holds its target instance, its protocol, an operation mask over `P`'s operations other than internal ones, and the credit of [Credits](#credits).

- The runner mints a port for each wire when it builds an Env, and for each `Port` grant that `grants` names. A wire mints its port only when the instance or the principal it acts for holds a grant covering the wire's target; a plan writing a wire that no grant covers is refused (`auth.denied`) ([Env construction](09-authority.md#env-construction)). Such a port is a **minted port**. A port derived from it by attenuation, or copied from it, shares its minted port. A wire's minted port lasts as long as the wire: an Env rebuilt with the wire unchanged holds the same minted port.
- A port names an instance, not a moniker. When the instance is removed, its ports never reach an instance created again under its moniker.
- `Port.only(operations)` attenuates: it returns a port whose mask holds only the operations it is given that the port's mask holds ([NaN 14 §Attenuation](https://github.com/nan-language/nan/blob/main/docs/spec/14-authority.md#attenuation)). Ports attenuate only by mask; an expiry is a scheduled revocation ([revocation](09-authority.md#revocation)).
- Changing or removing a wire, or revoking a grant, revokes its minted port, and with it every port derived or copied from that port, in every instance holding one.

## Wiring

`InstanceSpec.wiring` maps an Env field path to a `Wire` record ([instance specifications](06-execution.md#instance-specifications)):

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `target` | `Instance(Moniker)` or `Role(Role)` | required | the instance the port reaches, or the device role whose driver or volume view it reaches ([volumes and roles](15-devices.md#volumes-and-roles)) |
| `operations` | `Option[mem::List[text::String]]` | none: every operation that is not internal | the port's mask |
| `credit` | `Credit {messages: u16, calls: u16}` | 64 and 64 | the [credit](#credits) of the port, each count from 1 to 4,096 |

The Env field's type is `Port[P]`. `P`, as the instance's bundle declares it, is the wire's **client protocol**; the protocol the target serves is its **server protocol**. A plan writing a wire whose target instance has no specification is refused (`port.no-target`).

The **compatibility check** of a wire holds when its client and server protocols come from one bundle, when their protocol hashes are equal, or when each operation of the mask has equal operation hashes on both sides or satisfies all of these (requires [N4](22-nan.md#n4), [N20](22-nan.md#n20)):

1. The server protocol has an operation of the same name and kind that is not internal.
2. The server's argument schema tree accepts the client's ([NaN 29 §Encode class](https://github.com/nan-language/nan/blob/main/docs/spec/29-identity-versioning.md#encode-class)), and the client's result and failure schema trees accept the server's.
3. The server's reversibility bound is at most the client's.

The check runs when a plan writes a wire, when the runner builds an Env, and when either end changes bundle or component; it never runs per message. A plan whose wire fails it, or whose mask names an operation the client protocol lacks or keeps internal, is refused (`port.incompatible`); a start fails the same way ([starting](06-execution.md#starting)). Admission applies the check to the wires of every instance of a bundle it admits ([admission](10-code.md#admission)).

## Sends

| Operation | Called from | Arguments | Result | Failures |
| --- | --- | --- | --- | --- |
| `Cx.send_to` | a handler | a `Port[P]`, a one-way message of `P` | — | `port.full`, `port.revoked`, `port.stopped` |
| `Port.post` | a task | a one-way message | — | `port.revoked`, `port.stopped`, cancellation |
| `Port.call` | a task | a request | `R` | `E`, `port.revoked`, `port.stopped`, `port.server-stopped`, `port.malformed`, cancellation |
| `Cx.reply` | a handler | a `ReplyId[R, E]`, a `Result[R, E]` | — | `port.stale` |

- In NaN code each failure is the variant of the operation's error enum whose name is the code's in PascalCase (`ServerStopped` for `port.server-stopped`); `send_to` fails with `nanos::SendError {Full, Revoked, Stopped}` (requires [N3](22-nan.md#n3)).
- `send_to` never waits. It fails at once when the port's message credit is exhausted, when the port is revoked or its mask lacks the operation, and when the target is `stopped`, `failed`, removed, or `stopping` other than for a restart. The message is held until the handler returns ([held effects](06-execution.md#held-effects)); a revocation after the check does not withdraw it.
- `post` waits for credit, then returns once the message is in the target's intake. It is a cancellation point ([NaN 17 §Acknowledged cancellation](https://github.com/nan-language/nan/blob/main/docs/spec/17-suspension-io.md#acknowledged-cancellation)), acknowledged at once, and so is `call` (requires [N3](22-nan.md#n3)).
- A message to a `starting`, `running`, `hibernated` or `updating` instance, or to one restarting, waits in its intake; one to a `hibernated` instance revives it ([lifecycle](06-execution.md#lifecycle)).
- When the runner removes a message from the intake for a handler call, it decodes the message when encoded, adds a pending reply for a request and a received-capability entry for each capability argument, and returns a one-way message's credit.

## Calls

`call` waits for a call credit, puts the request in the target's intake, and waits for the reply. A cancellation removes the request from the intake, or ends its pending reply. A request ends without a reply in these cases:

| Case | The call |
| --- | --- |
| its server stops, restarts or changes bundle ([runner tables](06-execution.md#runner-tables)) | fails with `port.server-stopped` |
| its encoding does not decode at the server ([transfer](#transfer)) | fails with `port.malformed` |
| its calling task is cancelled | fails with the cancellation |
| its deadline passes | is cancelled by the caller's `task::within`; its outcome is `port.deadline` |

A request's deadline is the calling task's ([deadlines](08-scheduling.md#deadlines)); at that deadline the runner removes the request from the server's intake or pending replies. A request without a deadline waits until it is answered or ends otherwise.

## Replies

A request reaches its server as its operation's variant, with a `ReplyId[R, E]` naming a new entry of the server's pending replies: the calling task, the request's deadline, its cause and its account. The entry is charged to the caller's budget. The server answers with `Cx.reply` in that handler call or in any that follows, its state keeping the id.

The reply is held until the handler returns, then passes to the calling task as [transfer](#transfer) states, and the entry ends. A reply to an entry that has ended fails with `port.stale`.

## Credits

A **credit** bounds what one sending instance has in flight through one minted port: at most `messages` one-way messages waiting in the target's intake, and at most `calls` requests outstanding. It is `Wire.credit` for a wire, and 64 messages and 64 calls for a port a grant mints.

- A one-way message counts from its release until a handler call of the target takes it; a request, from its release until its call ends.
- A waiting message's memory is charged as [cells and limits](05-memory.md#cells-and-limits) and [transfers](05-memory.md#transfers) state.
- When the message credit is exhausted, `send_to` fails with `port.full` and `post` waits; when the call credit is exhausted, `call` waits.
- Messages an instance sends itself, through `Cx.send`, `Cx.after` and its outboxes, take no credit; their entries are charged the same way, and exhaustion traps ([NaN 26 §Component context](https://github.com/nan-language/nan/blob/main/docs/spec/26-components.md#component-context)).

An idle port costs nothing. A message costs one intake entry and two updates of its credit count.

## Ordering

- Messages one instance sends to another enter the target's intake in the order of their release, through whichever ports, and the target's handlers take them in that order. A handler's held effects are released in the order requested, a task's sends in program order, and the sends of concurrent tasks in the order the runner accepts them.
- No order holds between messages of different senders, nor between a reply and messages.
- The intake is one FIFO. Deadlines change when an instance runs, never the order of its intake ([deadlines](08-scheduling.md#deadlines)).
- A message keeps its place while its target is `starting`, `hibernated` or `updating`. A redelivered dead letter enters at the tail ([dead letters](06-execution.md#dead-letters)).

## Headers

Every message carries a 24-byte **header**, which the runner writes:

| Field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| `cause` | 0 | 8 | the [cause](17-observability.md#causes) of the handler call or task that sent it; bit 63 set when the cause is traced |
| `deadline` | 8 | 8 | the deadline of the handler call or task that sent it ([deadlines](08-scheduling.md#deadlines)), in monotonic nanoseconds since boot ([time](02-identifiers.md#time)), 2^64 − 1 for none |
| `account` | 16 | 4 | the [account](08-scheduling.md#accounting) its handling is charged to |
| `flags` | 20 | 4 | bit 0: a request; bit 1: an encoded payload; bit 2: capability arguments; bit 3: redelivered; bits 16 to 23: the operation's number; other bits 0 |

A keyed task sends with the cause and account of the handler call that spawned it; a job's `main` with the cause of its `run` step and the job's account. A handler call sends with those of the message it handles.

## Transfer

| Sender and receiver | Transfer |
| --- | --- |
| one bundle | the value moves, with no copy and no encoding; its chunks are charged as [transfers](05-memory.md#transfers) states |
| two bundles, one space | the sender's derived encoder writes the canonical `mem::wire` encoding ([NaN 13 §Wire format](https://github.com/nan-language/nan/blob/main/docs/spec/13-text-encode.md#wire-format)) into an intake buffer; the receiver's derived decoder decodes it, in the receiver's domain, when a handler call takes it |
| two spaces | the encoding travels through a [channel](#isolated-transport) |

The base instances form one bundle, so their messages move. A transfer between bundles costs one encode, one copy and one decode. Store blobs pass between instances as read-only views, without copy ([mapped blobs](05-memory.md#mapped-blobs)).

- A decoder of a message enum that is a parsing type is a parsing validator; any other is a declassifying one, a trust escape that admission treats as such ([NaN 15 §Validator classes](https://github.com/nan-language/nan/blob/main/docs/spec/15-untrusted.md#validator-classes)). `Untrusted` fields decode structurally and stay opaque (requires [N29](22-nan.md#n29)).
- A message that does not decode is discarded: a request fails with `port.malformed`, and the runner emits `port.undecodable`.
- When an instance changes bundle, the runner re-encodes the messages in its intake and its pending `after` messages with the old bundle's encoders, and the new bundle decodes them when a handler call takes them (requires [N4](22-nan.md#n4)).

## Capability transfer

A capability argument travels with its message: sending copies it, and delivery adds an entry naming the same capability to the receiver's received-capability table, whose `CapId[C]` the variant holds ([runner tables](06-execution.md#runner-tables)) (requires [N4](22-nan.md#n4), [N27](22-nan.md#n27)). Revoking the capability revokes the received copy. A capability in a reply reaches the calling instance's table the same way. Between bundles the capability travels beside the encoding; between spaces the core adds the entry, as the [isolated transport](#isolated-transport) states.

Only `Port`, `View`, `Control`, `Egress`, `Listen`, `WallClock` and `sys::Clock` capabilities travel ([capability kinds](09-authority.md#capability-kinds)). Admission refuses a protocol with an argument of any other capability type (`port.capability-kind`). No received capability persists: a checkpoint or a dead letter holds only its id, stale after a restart.

## Isolated transport

Messages between instances in different spaces travel through **channels** (requires [N23](22-nan.md#n23)). The core creates a channel for each pair of a sending instance and a minted port whose target lies in another space, at its first message, and frees it when the minted port is revoked and its outstanding calls have ended. A channel is 10 frames, mapped into both spaces and charged as [transfers](05-memory.md#transfers) states, a failed draw trapping the sender:

| Frames | Written by | Holds |
| --- | --- | --- |
| 0 | the core; read-only in both spaces | the channel's state, mask and credit |
| 1 | both sides | the indexes |
| 2 to 5 | the sender | the **forward ring**: one-way messages and requests |
| 6 to 9 | the receiver | the **reply ring**: replies and failed requests |

| Frame 0 field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| `state` | 0 | 4 | 0 open, 1 closed |
| `messages`, `calls` | 4, 6 | 2 each | the credit |
| `mask` | 8 | 32 | bit i set when the sender holds a live port of this minted port whose mask has operation i |

| Frame 1 field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| `forward_tail` | 0 | 8 | bytes the sender has written to the forward ring |
| `reply_head` | 8 | 8 | bytes the sender has consumed from the reply ring |
| `sender_waiting` | 16 | 4 | 1 while the sender sleeps until a doorbell |
| `forward_head` | 64 | 8 | bytes the receiver has consumed from the forward ring |
| `reply_tail` | 72 | 8 | bytes the receiver has written to the reply ring |
| `receiver_waiting` | 80 | 4 | 1 while the receiver sleeps until a doorbell |

Each ring is 16,384 bytes of records, at offsets taken modulo 16,384. A record never wraps: one that does not fit before the end of the ring follows a padding record that fills it.

| Record field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| `size` | 0 | 4 | bytes of the record, a multiple of 8 |
| `kind` | 4 | 1 | 0 padding, 1 one-way message, 2 request, 3 reply, 4 failed request |
| `reserved` | 5 | 1 | 0 |
| `caps` | 6 | 2 | the number of capability ids after the payload |
| `call` | 8 | 8 | for a request, a number its sender gives it, whose high 32 bits are the low 32 bits of the sender's incarnation; for a reply or a failed request, the number of the request; otherwise 0 |
| `header` | 16 | 24 | the [header](#headers) |
| `length` | 40 | 4 | the payload's bytes |
| `transfer` | 44 | 4 | 0 for an inline payload, otherwise the number of the core transfer holding it |
| `payload` | 48 | `length` when inline | the `mem::wire` encoding of the arguments, of the `Result[R, E]`, or for a failed request of its code as text |
| capability ids | after the payload, at a multiple of 8 | 8 each | the received-capability entries the core added for this record |

These rules govern a channel:

- **Payloads.** A payload of at most 4 KiB travels inline. A larger one travels through a core transfer (`transfer` and `accept`, [system calls](04-kernel.md#system-calls)), which copies it below 256 KiB and moves its frames from the sender's space into the receiver's from 256 KiB.
- **Capabilities.** For a message with capability arguments, the sender passes each capability to the core (`pass`), which adds its entry to the receiver's received-capability table, pending for this channel; the receiver claims the entries that the record lists (`claim`) when a handler call takes the message.
- **Credit.** The receiver copies a one-way message out of the forward ring only while fewer than `messages` of the channel's messages wait in its intake, and a request only while fewer than `calls` of the channel's requests are outstanding. Records beyond that wait in the ring. A new incarnation of the sender discards the replies to its earlier incarnations' requests.
- **Doorbells.** A side writes its indexes with release ordering and reads the other side's with acquire ordering. Before it sleeps (`wait`), a side sets its `waiting` field, issues a full fence and reads the other side's indexes again. After advancing an index, a side issues a full fence and rings the other side's doorbell (`ring`) only when that side's `waiting` field is 1; a woken side clears its own.
- **Revocation.** When the minted port is revoked or its target removed, the core sets `state` to closed. The receiver discards every forward record it has not consumed; the reply ring serves the outstanding calls.
- **Untrusted sides.** Each side reads the other's indexes and records as `Untrusted`, and copies a record out of the ring before decoding it. In a record from an isolated space, the receiver replaces `account` with the sending instance's account. A record or an index out of range, an operation outside `mask`, or a capability id that is not pending for the channel is handled as an integrity fault of the instance whose side wrote it ([faults](04-kernel.md#faults)).

A message through a channel costs a record write and a copy out, and a doorbell only for a sleeping side.

## Events

| Kind | Payload | Section |
| --- | --- | --- |
| `port.undecodable` | sender, receiver, operation, payload bytes | [Transfer](#transfer) |

## Codes

| Code | Meaning | Section |
| --- | --- | --- |
| `port.capability-kind` | a protocol has an argument of a capability type that never travels | [Capability transfer](#capability-transfer) |
| `port.deadline` | a call's deadline passed before its reply | [Calls](#calls) |
| `port.full` | the sender's message credit for the port is exhausted | [Credits](#credits) |
| `port.incompatible` | a wire fails the compatibility check, or its mask names an operation the client protocol lacks or keeps internal | [Wiring](#wiring) |
| `port.malformed` | a request did not decode at its server | [Transfer](#transfer) |
| `port.no-target` | a wire's target has no specification | [Wiring](#wiring) |
| `port.revoked` | the port is revoked, or its mask lacks the operation | [Sends](#sends) |
| `port.server-stopped` | the server stopped, restarted or changed bundle before replying | [Calls](#calls) |
| `port.stale` | a reply names a request that has ended | [Replies](#replies) |
| `port.stopped` | the target is `stopped`, `failed`, removed, or `stopping` other than for a restart | [Sends](#sends) |
| `port.too-many-operations` | a protocol has more than 256 operations | [Protocols](#protocols) |
