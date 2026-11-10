# 0048. View records

Status: accepted

## Context

Chapter 09 lets a View narrow to a subtree, to reading alone and to a commit. A capability's attenuations live in its record, which each derivation narrows and which NaN keeps, past two words, in a block each entry owns.

## Decision

- A View's record is 256 bytes: its rights, the commit it reads as of, and the path below its root entry's subtree, at most 240 bytes. `View.below` fails with `store.too-large` past that.
- A View narrowed to a commit reads alone: at its commit or an earlier one, `store.denied` for a later one and for subscriptions, `store.unversioned` for streams, and `store.kind` for counts and scans, which read the head only.

## Consequences

- Each View costs its entry a 256-byte block.
- A View narrows to a subtree only while the path below its grant's fits 240 bytes.
