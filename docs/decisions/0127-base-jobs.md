# 0127. Base jobs

Status: superseded by 0147

## Context

Chapter 20 has a generation ship its jobs as admitted bundles of provenance `vendor`. Admission refuses a bundle whose trust escapes carry no seal by a sealer the policy names, and a seal attests a human review. `selftest`, `scrub` and `replay` decode records holding text and bytes, whose decoders are trust escapes. The genesis record carries the base bundle's admission record, which the running kernel builds (0122). The boot volume holds the loader's and the kernel's images only.

## Decision

- **In the kernel image.** `selftest`, `scrub` and `replay` are library packages the kernel image links, each exporting the `main` of its job.
- **Code hash.** A base job's code hash is that of the `CodeId` rooted at its name whose one code root is the base bundle's code hash.
- **Admitted at genesis.** The genesis record holds each base job's admission record, which the kernel builds as the base bundle's: provenance `vendor`, shared, the kernel image as its image, and its `main`'s input and output schemas and class. Genesis writes them, and a later boot writes those `main` lacks.
- **Run from the image.** The runner asks its components whether a code hash is a base job's, and runs such a job's `main` from the kernel image in the job's domain, decoding its input and encoding its result as a loaded bundle's.

## Consequences

- A base job changes only with a new generation.
- The `nan` job, being the toolchain built as its own program, is no base job.
