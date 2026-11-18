# 0122. Base admission record

Status: accepted

## Context

Chapter 10 gives the base bundle an admission record from the generation's genesis record, and `selftest` checks that every specification's bundle is admitted. The kernel image holds its genesis record in code. Most of an admission record comes from a bundle's artefacts: its packages with their code roots and authors, each operation's digest line, each component's Env fields, the closure summary and trust escapes. NaN writes no artefacts for a bare image (nan-language/nan#256), and the kernel cannot read the toolchain hash its interface file records. A later boot may run a kernel image of another code hash than the one genesis recorded.

## Decision

- **Built by the running kernel.** The kernel describes each base component from its own tables: its state schema, protocol hash, operations with their declared bounds and configuration type. The record names provenance `vendor`, shared placement, and the kernel image as its image, of the blob id the loader hashed, the support routines it exports and the bytes it spans, lowered at the base set of x86_64. What only artefacts tell is empty, and the toolchain hash is all zeros.
- **Written with the base instances.** Genesis writes `/sys/code/<hash>/admission` in commit #1; a later boot's generation commit writes it when the running kernel's code hash has no dir there.

## Consequences

- Updating a base instance compares the new bundle's component with the record's state schema and protocol hash.
- The record lacks packages, digest lines, Env fields and the toolchain until a generation's genesis record is built from its kernel image's artefacts.
