# 0033. The store engine with head reach

Status: accepted

## Context

Chapters 11 and 12 define the store: one sequencer per store taking transactions in order, a log of batches whose flushes make commits durable, copy-on-write B+trees reached through an item table, retention, thinning, a cleaner and scrub. Its engine runs, and is tested on the host, against an in-memory volume, before the store's instance, protocol and placement exist. Thinning and `LiveStore` need the bytes only past commits reach and the bytes agents added, while branches share trees, nodes and streams from their fork. `expect` is a NaN keyword.

## Decision

- **Package.** `store` targets `any`: `disk` holds the device interface, the in-memory volume and the records' layouts, `log` the segments and batches, `tree` the B+trees and the item table, and the root module the engine, one `Store` value whose operations take the device as a `disk::Device` parameter, for the sequencer of the store's instance to call. XXH3-128 lives in `ident`, which `bootfmt` and `store` share.
- **Commit procedure.** Writes apply to the branch's trees as the procedure checks them, each recording what it replaced, and a failing check undoes them all, so a write sees the transaction's earlier writes and a refused transaction leaves nothing.
- **Group commit.** A checked commit is numbered, logged and sealed; one log flush makes every sealed commit durable, or, failing, undoes them newest first, each failing with `store.io`. `commit_all` commits a batch through one log flush, splitting it where an index flush is needed, since an index flush never runs while a commit is sealed. An index flush runs before a transaction applies writes whose entry bytes would leave more than 16 MiB of log after the latest flush record.
- **References.** The node entries of a retained commit's record hold references to what they name until the commit is thinned, so a value a later commit of the same flush interval replaced stays readable at its commit.
- **Head reach.** Beside its reference count, each item, blob and stream counts the references from what branch heads reach: their roots, and the nodes those reach. A change to a head's tree mirrors its reference changes into these counts; at an index flush, a node heads come to reach, or cease to, adds or removes one such reference to its children and values, until nothing changes. Branch creation stays O(1), and a flush costs what changed. History bytes are what something names but no head reaches; agent bytes are the values, blobs and stream runs agents added, an agent's record always being a value record so that none hides in a shared leaf.
- **Space.** The engine keeps 2 segments plus 0.5% free. An agent's write past `agent_pct` of the capacity, or lacking room, fails with `store.full`; another principal's write lacking room thins unpinned commits other than heads from the oldest, in doubling rounds each followed by cleaning and index flushes. An index flush thins in doubling rounds while history bytes, as settled so far, pass `max_pct`.
- **Streams.** A commit changing a stream another head holds at the same path gives its branch a stream of its own, the runs copied; a stream's runs go when no head reaches it.
- **Past commits.** A commit is read through the tree of the latest flush commit before it and the commit records since. Before thinning a commit a kept one reads through, thinning gives the kept one a tree of its own; a head gets its tree from the index flush.
- **Records.** A commit record holds the catalog entries the commit writes besides its own; a request entry keeps its commit's branch and time and lasts as long as the key is among its session's latest `key_results`; the flush record keeps the preview counter and the history and agent bytes; the transaction's expectations are `expects` and `draft_close` takes `expected`. Chapters 11 and 12 state these.

Alternatives rejected:

- **Checking a transaction against an overlay before writing the trees:** every check would read two layers, and replay would need the overlay too.
- **Walking the heads' trees to measure history:** each flush would cost the size of the store.
- **Run records shared by diverged streams:** they would need counts of their own; a copy costs only the stream's retained entries.

## Consequences

- Values that became history since the last index flush count from the next one, so history can pass `max_pct` by one flush interval.
- Stream retention on a stream several heads share applies to all of them.
- Diverging on a large unbounded stream copies all of its retained runs.
