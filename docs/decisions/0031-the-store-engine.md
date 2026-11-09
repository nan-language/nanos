# 0031. The store engine

Status: accepted

## Context

Chapters 11 and 12 define the store: one sequencer per store taking transactions in order, a log of batches whose flushes make commits durable, copy-on-write B+trees reached through an item table, retention, thinning, a cleaner and scrub. The store's instance, its protocol and its placement come later; its engine must first run, and be tested on the host, against an in-memory volume. `expect` is a NaN keyword. NaN gives every temporary its own stack slot, so frames grow with call sites ([nan-language/nan#127](https://github.com/nan-language/nan/issues/127)), and decoders name no field path ([nan-language/nan#130](https://github.com/nan-language/nan/issues/130)).

## Decision

- **Package.** `store` targets `any`: `disk` holds the device interface, the in-memory volume and the records' layouts, `log` the segments and batches, `tree` the B+trees and the item table, and the root module the engine, one `Store` value whose operations take the device as a `disk::Device` parameter, for the sequencer of the store's instance to call. XXH3-128 moves from `bootfmt` to `ident`, which both use.
- **Commit procedure.** Writes apply to the branch's trees as the procedure checks them, each recording what it replaced, and a failing check undoes them all, so a write sees the transaction's earlier writes and a refused transaction leaves nothing.
- **Group commit.** A checked commit is numbered, logged and sealed; one log flush makes every sealed commit durable, or, failing, undoes them newest first, each failing with `store.io`. `commit_all` commits a batch through one log flush, splitting it where an index flush is needed, since an index flush never runs while a commit is sealed.
- **Log bound.** An index flush runs before a transaction applies writes whose entry bytes would leave more than 16 MiB of log after the latest flush record.
- **Space.** The engine keeps 2 segments plus 0.5% of them free. When a write lacks room, a principal other than an agent or a child principal has unpinned commits other than heads thinned from the oldest, in rounds of 1, 2, 4 and more commits, each followed by an index flush, cleaning and two more index flushes, and its transaction is then checked again.
- **References of commit records.** The node entries of a retained commit's record hold references to what they name until the commit is thinned, so a value that a later commit of the same flush interval replaced stays readable at its commit.
- **Past commits.** A commit is read through the tree of the latest flush commit before it and the commit records since. Before thinning a commit a kept one reads through, thinning gives the kept one a tree of its own; a head gets its tree from the index flush.
- **Records.** A commit record holds the catalog entries the commit writes besides its own, which replay writes again; a dir's entry holds the number its next `Add` names; a request entry's session is optional, an instance's commit having none; the transaction's expectations are `expects` and `draft_close` takes `expected`. Chapters 11 and 12 state these.

Alternatives rejected:

- **Checking a transaction against an overlay before writing the trees:** every check would read two layers, and replay would need the overlay too.
- **Flushing the index after applying a transaction's writes:** the flush would make durable trees that no commit record explains.
- **Thinning every unpinned commit when space lacks:** it drops more history than the write needs.

## Consequences

- Agent bytes and history bytes are not counted: `agent_pct` and `max_pct` have no effect, `store.history-thinned` reports 0 bytes and `store.writes-refused` 0 history bytes.
- A failing `schema.invalid` names its `core::DecodeError` without a field path.
- Store calls take several kilobytes of stack each; the store's instance needs a stack well above the 64 KiB default.
