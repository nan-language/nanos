# 0168. TPM commands apart from the driver, and keys sealed to the TPM

Status: accepted

## Context

The `tpm-crb` driver serves `Tpm` through the CRB interface; the loader reaches the same TPM through the firmware's `SubmitCommand`. Both speak TPM 2.0 commands. `keys` keeps its keys encrypted under a key it seals with the TPM, bound to no PCR, while its Env is rebuilt at every start and its state is its checkpoint.

## Decision

- **One `tpm` package.** The command builders and the response validators live in a package targeting `any`, apart from the driver, so that any program reaching a TPM sends the same commands. It knows nothing of how they reach the TPM.
- **Seals under a regenerated storage key.** `seal` and `unseal` create the owner hierarchy's ECC P-256 storage key from a fixed template each time, so no persistent handle is needed. A seal binding PCRs takes its policy digest from a trial session, so no SHA-256 runs outside the TPM. The sealed blob is the PCR selection and the object's sized public and private areas.
- **A step machine, polled.** The driver runs each operation as its commands in turn, one on the TPM at a time, flushing every handle it made, and polls completion with delayed messages to itself.
- **Keys at rest, keys in use.** The checkpoint of `keys` holds its keys in plaintext or encrypted under a sealed 256-bit key. The plaintext keys live in its Env, which a start fills from the checkpoint, unsealing through the `tpm` role. A seal completes before the checkpoint takes the new keys, and no secret record is sealed before then.

## Consequences

- Each seal or unseal costs a primary key creation, tens of milliseconds on a TPM.
- A TPM the driver times out on keeps the handles its operation made until the TPM restarts.
- `keys` serves nothing that needs its keys between its start and the unseal's end.
