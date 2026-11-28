# 0171. Volume encryption in the log, converted by the cleaner

Status: accepted

## Context

Volume encryption seals every log record and head-area block but the superblocks and headers under a key the TPM seals, and converts a plain volume in place. The engine is target-independent and draws no randomness; its index reads records with only the device in hand; its segment entries in the catalog are a fixed wire record; and a sealed record is 40 bytes longer than its plain form, so its layout is fixed when the log places it.

## Decision

- **The log seals as it places.** The volume key lives in the engine, the log and the index once the mount or `encrypt` gives it; every reader opens a record whose flag bit marks it sealed, and a plain one as before, so a converting volume reads both. A sealed volume aligns no payload to a block.
- **Nonces from a count.** The store draws 16 random bytes at each mount; each nonce is those bytes and the next count, so nonces never repeat on a volume without drawing randomness per record.
- **The cleaner converts.** `encrypt` marks every segment holding records plain, ends the open segment, and rewrites the keyring and a flush record sealed; while converting the cleaner takes plain segments first whatever their live data, and once none remains the superblocks mark the volume encrypted. A segment's plain mark is a high bit of its catalog state code.
- **The key passes through `Sealing`.** `keys` makes and unseals the volume key through its TPM port as two more operations of the protocol through which the store already reaches it.

## Consequences

- A converting volume is written sealed at once, and becomes encrypted only after the cleaner has moved every plain segment's live records.
- A store without `keys` or a TPM cannot mount an encrypted volume: `store.locked`.
