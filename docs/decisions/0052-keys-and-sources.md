# 0052. Keys and sources

Status: accepted

## Context

Chapter 09 has `keys` generate the link, audit and keyring master keys at genesis from the kernel's entropy and, without TPM sealing, keep them in plaintext in its checkpoint, write the principal records of enrollment, rotation and revocation, and verify signatures for any instance holding a port to it; `/live/security` reports `key_storage: software` and the link key's public halves, which `keys` provides beside the core. Chapter 20 starts `keys` before `store` without reading its checkpoint, then has the runner restart it from its checkpoint once `store` runs, and has each instance emit its events through `journal`'s `emit`, the source being the emitting instance. Chapter 19 has `keys` rewrite `/sys/principals/owner` to the loader's owner key at each boot when it holds another, emitting `policy.owner-mismatch`. Calls carry no header naming their sender. NaN's `core::crypto` signs and verifies Ed25519 and has no X25519.

## Decision

- **Sources.** The runner gives each instance's world a `Source`, a capability on `journal`'s port naming the instance and the principal it acts for, under the key `events`. `emit` carries its source, from which `journal` reads the emitting instance. `journal` refuses an event of a family of the core and the base instances from another source than a base instance, emitting `journal.kind-refused`, drops a source's events past 1,000 in a second, emitting `journal.lost` with the count when its next second begins, and holds the events it takes for at most 100 ms or until they reach 1 MiB, a `critical` one appended at once.
- **Unkept starts.** A checkpointed instance starting while `store` takes no calls starts without its checkpoint and keeps none. `supervisor` restarts a base instance that a later step of the start order names again, which step 6 does for `keys`.
- **Keys.** `keys` is checkpointed. Once its checkpoints are kept, it generates the machine's keys when its state holds none and checkpoints them at once. It serves `verify`, `enroll`, `issue`, `rotate` and `revoke` through Views of `/sys/principals` and `/data`, answering each request that reads or writes the store from a keyed task. `verify` checks the purpose's signers the chapter fixes, leaving to their readers those the policy names.
- **Security.** `keys` publishes its part of `/live/security` at `/live/security/keys`, which the board merges into the security record and lists nowhere.

Alternatives rejected:

- **A source named in the call:** an instance could emit as another.
- **Failing a checkpointed start while `store` takes no calls:** `keys` could not start before `store`.

## Consequences

- An event emitted through `journal` acts for `system` under the boot's system cause, as the core's events do.
- `keys` started before `store` serves nothing: no volume is encrypted.
- `/live/security` names no X25519 half of the link key until NaN gives X25519.
- `enroll` writes the principal record and `/data/<principal>` in two commits.
