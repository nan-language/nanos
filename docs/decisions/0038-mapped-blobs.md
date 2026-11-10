# 0038. Mapped blobs

Status: accepted

## Context

Chapter 05 has a mapped blob read in place: its 64 KiB pieces lie once in the store's blob cache, whatever the number of readers, which reach them through read-only views, and a piece stays while a mapping holds it. Chapter 11 has `map` give such a mapping. A capability kind runs its methods on the caller's task with the state of its root entry, which lives in the minting domain and goes when the entry closes; a shared value tells no count of its holders. The store postpones every request and answers it through the runner, whose replies carried no handles.

## Decision

- **Blob cache.** The engine holds each piece it reads by its key, shared, and lets the least recently used go once it holds more than 8 MiB. Reads go through it.
- **Mappings.** `map` gives a `Mapping` capability whose state holds shares of the pieces of its range, and `Mapping.read` copies a part of the range out of them. The store holds the root entries of its latest 64 mappings, revoking the oldest when it makes another.
- **Handles in replies.** The runner hands the driver the handles beside a postponed reply, and the store encodes a mapping in its reply as the index of its handle.

Alternatives rejected:

- **Giving readers the shared pieces:** they would outlive the store's domain, whose memory holds them.
- **Keeping the pieces mappings hold by counting holders:** a shared value tells no count.

## Consequences

- A mapping goes stale once the store has made 64 later ones, and when the store restarts.
- The cache's pieces are charged to the store's cell, and pressure does not drop them.
- A piece the cache let go while a mapping holds it is read again for a later mapping, held twice meanwhile.
- No reader in an isolated space has a mapping in its window, and `map` takes no `at`.
