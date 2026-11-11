# 0065. Link connections and session authority

Status: accepted

## Context

Chapter 18 serves the link on a port transport too: a resident agent opens a connection with `LinkSession.open`, sends request lines with `send`, which takes no connection, receives each block as a `LinkClient.block`, and opens its sessions as `inst.<moniker>`, the principal of the instance that called `open`. A handler sees the header of the call it serves, whose spare field the runner left unused, and the core alone knows which instance each slot of the runner's table holds. Chapter 09 lets a principal open sessions only under a live `Session` or `Machine` grant, from which its sessions take their budget and with which they read; a session ends when that grant goes or its principal's key is revoked, and detaches when its connection is lost. The link reads the tree through one View of `/`, and its handlers never wait. Requests take `--fields` and `--json`, whose records `text::json` would encode from typed values the link only holds as canonical renderings. The specification's transcripts show a machine in `normal` mode with enrolled agents and installed applications, which the sandbox, in `dev` mode with the base instances alone, does not have.

## Decision

- **Callers.** The header of the work handling a call names the slot of the instance that sent it, and `os::Callers`, a capability the core mints over its table of the instances' monikers, names the instance of a slot. `link` holds it.
- **Connections.** The link keeps one connection per transport stream: the serial console's, and one per instance on the port transport, a second `open` of the same instance losing its first. Each connection has its blocks, its writer, its challenge and the request it gathers; a lost connection detaches its sessions.
- **Authority.** At `hello` a keyed task reads the principal's grants and whether its record holds a key; the link keeps them for the principals holding sessions and reads them again on each change its subscriptions of `/sys/grants` and `/sys/principals` tell, ending each session they no longer let go on. Before a request runs, the link checks that the session's grants cover each path it reads, and fails it with `auth.denied` naming `tree.read` otherwise.
- **Budgets and taint.** The link counts output units and keeps the taint per principal, so its sessions spend and gain together; a block's `taint` record and its units are computed when the handler renders it.
- **Notices.** Session events, rejected `hello`s and the records of `/live/sessions` go through one queue that one keyed task empties in order.
- **Output.** `--fields` narrows a record's header, its hashes in full forms when it names `hash`; `--json` converts each record from its canonical rendering, constructions as objects, variants as their names or `{"Variant": payload}`.
- **Transcripts.** `link/transcripts` holds one transcript per transcript of chapter 18 and per verb, in the specification's form on the sandbox's state, `*` standing for what differs between boots; `tools/transcripts` runs them on the serial console of a fresh sandbox.

Alternatives rejected:

- **A connection argument to `send`:** the specification's `send` takes the line alone.
- **Capabilities minted per session:** the link would need authority to mint from other principals' grants, which only the runner holds.
- **The specification's transcripts verbatim:** their machine state cannot exist in the sandbox before plans, code and enrollment do.

## Consequences

- The CPU, memory, egress, energy and inference budgets wait for sessions' accounts, and `previews` for plans; `describe` of protocols and types waits for the APIs that admitted bundles carry, and `history` of a moniker for flight recorders.
- A session's reads are as narrow as its grants, while the link's own View stays wide.
- A transcript depends on the sandbox's base instances and genesis records, and changes with them.
