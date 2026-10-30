# 0011. Bundles, admission and live update

Status: accepted

## Context

An AI controlling the machine changes its code constantly: drivers, services, kernel policies. NaN defines no ABI between separately built final closures, and lowered code belongs to one image. NaN's reuse of an artefact keeps its recorded proof statuses, so an artefact claiming a proof it never made would ship without its run-time check. Small devices cannot run the toolchain.

## Decision

- The unit of loading is the **bundle**: one lowered final closure of one or more component packages, or of one program, in a NaN OS file holding:
  - the artefacts;
  - lowered images per CPU feature level;
  - an optional check certificate;
  - signatures.
- Values move only between instances of one bundle. Between bundles, and between spaces, messages are `mem::wire` encodings checked by the receiver's derived decoder. Bundles therefore update independently, and no layout couples them.
- Admission verifies:
  - **checking:** a device with the toolchain re-checks the closure in verify mode; a device without one requires a certificate from a builder the policy trusts;
  - signatures and signed seals;
  - the authority summary against the requested grants;
  - requires-sets;
  - state schemas and protocol hashes;
  - reversibility, provenance and placement.
- A live update pauses an instance's intake at quiescence, re-encodes its state through the new code, and keeps the old bundle through a probation window. During probation the flight recorder keeps full inputs, so a revert replays them through the old code.
- The kernel image changes only by a generation switch (0010).

Alternatives rejected:

- **One closure for the whole machine:** every install would relink and reboot.
- **A shared value ABI across bundles:** every toolchain update would have to switch every bundle together.
- **Trusting recorded proofs:** undefined behaviour in the shared space.
- **Native code from publishers without a re-check or a certificate:** the boundary would move to someone else's compiler.

## Consequences

- A message between bundles costs an encode, a copy and a validating decode. The base instances form one bundle, so their traffic moves values.
- Admission needs verify mode ([N26](../spec/22-nan.md#n26)), loading ([N6](../spec/22-nan.md#n6)), schema trees ([N20](../spec/22-nan.md#n20)), `migrate` and `flexible` ([N5](../spec/22-nan.md#n5)) and signatures ([N15](../spec/22-nan.md#n15)).
