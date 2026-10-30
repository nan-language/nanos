# 0006. One typed, versioned store and one tree

Status: accepted

## Context

Text configuration files, byte-stream files, text logs and a global path tree make an agent parse, guess and spend tokens. A global namespace is also ambient authority. Configuration spread over files drifts, and nothing records who changed it or allows an atomic change across code, configuration and authority. NaN gives every `Encode` type a canonical encoding and a schema hash.

## Decision

- One store per machine holds four node kinds: records (typed values with their schema hash), blobs (bytes), dirs (typed or untyped maps) and streams (append-only typed sequences).
- A write is accepted only when the value decodes under its node's declared type, so invalid configuration never commits.
- History is kept for `/sys` and `/data`. Every commit names its principal, session, cause, plan and intent. Reverts are new commits. Branches are O(1), and merges are structural, per record field.
- Snapshot isolation is the default. Commits touching `/sys`, and plan commits, are serializable.
- The tree joins the store's nodes with `/live`, the observed state that instances serve. Desired state lives in `/sys`, and instances reconcile `/live` to it.
- Names never confer authority: a path resolves only inside the capability presented. `sys::Fs` is a view of a dir, so NaN programs keep their file API.
- The on-disk format is a checksummed log with a copy-on-write index, recovering by replaying at most 16 MiB of log.

Alternatives rejected:

- **A byte-stream file system beside a package database and text configuration:** three histories that disagree.
- **Serializable isolation everywhere:** it validates read sets of checkpoints and file writes that are single-writer.
- **A cryptographic root hash per commit:** about 260 µs of BLAKE3 per commit without SIMD.
- **Leases between agents:** locks in a lock-free design, and a channel leaking holders across principals.

## Consequences

- An agent reads and diffs configuration, code, authority and data with one query language and one notation.
- Every change can be reverted until retention thins it. Thinning is an erase, which the owner approves.
- Writing records checks predicates in the writer's typed code, or in the declaring bundle's exported decoder ([N20](../spec/22-nan.md#n20)). Keeping fields that older code does not know needs [N5](../spec/22-nan.md#n5).
- The sandbox's only disk is the FAT boot volume, so the store runs there as a preallocated file; a block device in NaN's descriptor is [N22](../spec/22-nan.md#n22).
