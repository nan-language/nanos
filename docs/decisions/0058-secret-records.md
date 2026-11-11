# 0058. Secret records

Status: accepted

## Context

Chapters 11 and 12 store a secret record as an envelope that XChaCha20-Poly1305 seals under a node key, which `keys` derives with BLAKE3 from the keyring master key, the seed of the record's keyring slot and its secret id; the store validates the plaintext, never holds a node key, and writes the slot's keyring block, both copies, before the commit holding the record is durable. NaN's `core::crypto` holds only Ed25519; `core::Secret` computes in constant time. `keys` depends on `store`, so `store` cannot name `keys`' protocol.

## Decision

- **Ciphers.** A `crypto` package holds XChaCha20-Poly1305 sealing and BLAKE3 key derivation, computing on `core::Secret` values.
- **Sealing protocol.** `store` declares `Sealing`, whose `seal_secret` `keys` serves within its own protocol; genesis wires `store` to `keys` with that operation alone, its `KeyUse(node)`.
- **Engine.** A commit takes the `Secrets` it seals through: the engine validates a secret write's plaintext, writes a new secret's keyring block, copy 0 then copy 1 each followed by a flush, has the value sealed, and stores the envelope. A commit that fails erases the secrets it made, and replay and the keyring number new secrets past every id they hold.
- **Reads.** A secret record gives its plaintext's size and schema, no value; one whose slot is gone fails with `store.erased`.

## Consequences

- Nothing opens an envelope: `keyring_seed` and the opening of secret records serve instances in isolated spaces, which do not run yet.
- Each mount reads both copies of the whole keyring, 1,982 blocks.
