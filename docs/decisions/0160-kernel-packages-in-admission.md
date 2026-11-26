# 0160. The kernel's own packages in admission

Status: accepted

## Context

Admission checks a bundle's closure in verify mode and needs every trust escape outside the toolchain's std packages sealed by one of the policy's sealers. A component holding NaN OS capabilities carries `os` and `ident`, the packages defining them, which the running kernel's interface file lists with their code roots; a bundle build against that file refuses other code roots for them. They pass no release gate: they keep no API snapshots, and the `capability` grant of `os` carries no seal. A kernel image holds its interface hash, not its interface file, and step 6 runs on every machine, whether it runs the `nan` job or not.

## Decision

- **The kernel's own packages.** The packages of a bundle's closure that the running kernel's interface file lists, with the code roots it records, are the kernel's own: admission judges them by those code roots alone, outside the release gate, the seals of step 6 and the reviews of provenance, as it judges the std packages of its toolchain.
- **The interface file on every machine.** A generation ships its kernel image's interface file beside the image, the loader passes it as a module, and the kernel takes it when its hash is the image's interface hash. The core gives it through a `code::Interface` capability, which admission and the toolchain's runs read.

Alternatives rejected:

- **Sealing `os` and `ident` by a sealer the policy lists:** a review identity none of their authors holds, and seals lapsing at each kernel change touching them, for code the machine already runs as its kernel.

## Consequences

- A bundle carrying the kernel's packages fails admission's check from its sources until N66, and passes on a certificate on a machine without the `nan` job.
- A kernel without its interface file judges its packages as any other.
