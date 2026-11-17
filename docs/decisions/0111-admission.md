# 0111. Admission

Status: accepted

## Context

Chapter 10 admits a bundle file at the preview of `install`, from its artefacts alone, on a machine that runs no toolchain. NaN signs artefacts in its own forms: a publisher signature over NaN's artefact message, NaN's own certificate, and seals over NaN's seal message, each with the public key beside it, and a provenance section naming the authors. No artefact part holds a signature of a NaN OS object hash. Without the toolchain, a device cannot tell whether a seal is current, since that needs the item and closure hashes of the sealed item. A NaN final build stops on errors but not on incompleteness, so a bundle build succeeds with lapsed seals; the release gate, `nan verify`, rejects them. A `dep` directive names a source on the building machine, while a requires-set names each item the package uses by canonical path, whose first segment is its package's name. A bundle build fails unless every package of its closure has the code root that the kernel's interface file records. The blob an `install` step names is one the session reaches, and the store reads blobs by path.

## Decision

- **NaN's signatures.** A package's `review` signature is its artefact's publisher signature, and a `seal` signature is NaN's seal signature: each signs NaN's message. Admission verifies each by the key beside it, and `keys` verifies that this key is the signer's current key, or a former one for the bundle held at `/sys/code/<hash>/bundle` (`verify_key`).
- **Currency by certificate.** A check certificate states that its builder ran the release gate on the closure and found no error and no incompleteness, so every seal it holds is current. Admission accepts a seal whose signature verifies, by one of the policy's sealers, and relies on the check of step 4 for its currency.
- **Closure by requires-sets.** The packages a closure needs are those its requires-sets name.
- **Requires-sets by interface.** An image whose support hash is the running kernel's was lowered against the interface file that records the kernel's releases, so its closure's requires-sets hold: step 5 holds for every image step 13 takes, and fails with no code of its own.
- **Std.** The std packages are those NaN's reserved std aliases name and its platform packages; their trust escapes need no seal.
- **The bundle as a View.** `admit` takes a read-only View of the bundle blob, which the link narrows from the session's View, so admission reads only what the session reaches.
- **Trust.** `policy` gives the reviewers, sealers and builders of the policy it decides by, and `agent_shared_space` (`trust`); admission matches the principals against them.
- **Image.** Admission uploads the image it takes as a draft blob; the plan's commit references it at `/sys/code/<hash>/image`.

Alternatives rejected:

- **NaN OS signatures in artefacts:** NaN writes its own forms, which carry the key, so binding that key to the principal record suffices.
- **Admission reading a blob by id:** a blob id carries no authority, so admission would read blobs no session reaches.

## Consequences

- A builder certifies only closures that `nan verify` accepts.
- A bundle the `nan` job does not check is only as sound as its builder's release gate.
- The store drops an image's draft an hour after its upload, so an `install` applies within the hour after its preview.
