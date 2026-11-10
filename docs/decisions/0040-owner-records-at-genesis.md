# 0040. Owner records at genesis

Status: accepted

## Context

Chapter 13 has genesis write the vendor principals' records and, on a claimed machine or in `dev` mode on an unclaimed one, the owner's principal record, grant and `/data/owner`, the record holding the owner key the handoff record names or none. Chapter 09 gives the owner's grant: holder `owner`, parent `g0`, object `Machine`. A principal record names no principal, and the runner holds the grants of the image's genesis, numbered as `/sys/grants` numbers them.

## Decision

- **Principals by name.** The genesis record holds the principals' records by name.
- **Owner.** When the handoff record holds an owner key, or the boot is in `dev` mode, the kernel adds to its genesis record the owner's principal record, holding that key or none, and `/data/owner`, and to the grants of the image's genesis, which the runner holds too, the owner's grant with every right after the base instances' grants.

## Consequences

- No vendor principal has a record while generations carry no vendor keys.
