# 0108. Bundle files

Status: accepted

## Context

Chapter 10 admits code as a bundle file: one NaN artefact per package of a closure, std included, the images lowered from them, an optional check certificate and bundle signatures, named by the code hash of its closure. A NaN bundle build writes the image, the artefacts of its final closure and their code roots' hash beside it, against a kernel's interface file, and checks the whole closure as a final build does. A machine without the `nan` job admits a bundle only under a certificate by a builder its policy trusts. Hosts lack a BLAKE3 tool, and NaN OS's records are `mem::wire` encodings.

## Decision

- **Records.** The `code` package holds the records of the bundle file and the code hash, for the programs that make bundle files and for admission.
- **Making.** `bundler`, a NaN host program, makes a bundle file from what one bundle build wrote: the artefacts in byte order of code root hash, and one image, of the codegen level and toolchain the kernel's interface file names and the interface's hash as its support hash.
- **Certifying.** It certifies the bundle as a builder whose Ed25519 key a 32-byte seed file holds, created when missing. It certifies only the outputs of a bundle build, which fails on any error of the closure.

Alternatives rejected:

- **A shell script over host tools:** no host tool computes BLAKE3 or writes `mem::wire` encodings.
- **NaN's bundle signature (`nan build --as`):** it signs NaN's own format with NaN's keys, not a NaN OS bundle signature or certificate.

## Consequences

- A bundle file holds its closure's std artefacts: about 5 MB for a job using std alone.
- A builder's seed file is its secret key.
