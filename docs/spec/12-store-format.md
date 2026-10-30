# Store format

This chapter defines how the store lies on a volume: conventions, placement, the volume layout, superblocks, pointers, segments and log batches, log records, items, the index trees and their nodes, the catalog, blob pieces, group commit, index flushes, mounting and recovery, reference counting, the cleaner, discards, scrub, the encryption of secret records and of volumes, and format versions and upgrades. The model it stores is [11-store.md](11-store.md); its failure codes are in [11-store.md](11-store.md#codes).

## Conventions

- Integers are little-endian, except in the keys of the index trees, which are big-endian so that byte order is numeric order.
- A block is 4 KiB; a segment is 8 MiB, 2,048 blocks.
- A **checksum** is XXH3-128 with seed 0 and the default secret, stored as 16 bytes: the low 64 bits, then the high 64 bits.
- Hashes are NaN hash values; BLAKE3 is BLAKE3-256; "wire" is `mem::wire` ([NaN 13 §Wire format](https://github.com/nan-language/nan/blob/main/docs/spec/13-text-encode.md#wire-format)). A pointer inside a wire record is its 32 bytes as `mem::Bytes`.
- Reserved and padding bytes are zero when written and ignored when read.
- The store reads every structure from the device through a validator ([NaN 15 §Validators](https://github.com/nan-language/nan/blob/main/docs/spec/15-untrusted.md#validators)). A structure failing its magic, checksum or decoding is corrupt: the read fails with `store.corrupt` and the store emits `store.corruption`.

## Placement

The store reads and writes its volume through the `Block` port of the `store-volume` role ([15-devices.md](15-devices.md#volumes-and-roles)), which resolves, in this order, to:

1. A GPT partition of type GUID `e4cad23c-fec4-40f8-8d0c-1161d07842d6`, named `nanos store`, of at least 64 MiB: the one on the device holding the boot volume, else the lowest in bus path order. `/live/store` lists the others, which the store never writes.
2. The file `/nanos/store.img` on the boot volume ([03-boot.md](03-boot.md#boot-volume)), through the view `devices.store-img` over its contiguous clusters. When the file is absent, `devices` has `fat` ([15-devices.md](15-devices.md#the-fat-component)) allocate it with half the boot volume's free space, rounded down to a multiple of 8 MiB and capped at 4 GiB − 8 MiB; under 64 MiB, no volume exists.

The **volume** is that range, its size rounded down to a multiple of 8 MiB; a store block is 4 KiB whatever the device's block size. A volume on which neither superblock holds the magic `NANOSST1` is formatted: both superblocks written, the head area zeroed, an empty index flushed and every segment discarded. Without a volume, mounting fails with `store.no-volume`.

## Volume layout

| Blocks | Holds |
| --- | --- |
| 0 | superblock A |
| 1–64 | the flush ring: 64 index flush records |
| 65 | superblock B |
| 66–1,056 | keyring, copy 0 |
| 1,057–2,047 | keyring, copy 1 |
| 2,048·s to 2,048·s + 2,047 | segment s, for s from 1 to the number of segments |

Blocks 0 to 2,047 form the **head area**, the only blocks the store writes in place. On a zoned block device the head area lies in conventional zones, no segment crosses a zone boundary, the segments of a zone are written in order, and a zone is reset only when all its segments are free; superblock flag bit 1 marks such a volume.

## Superblocks

The store writes superblock A, then B, at format, at an encryption change and at a format upgrade. Mounting takes the valid copy with the higher `sequence`.

| Field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| magic | 0 | 8 | `NANOSST1` |
| sequence | 8 | 8 | incremented at each superblock write |
| format | 16 | 4 | the volume's format version |
| flags | 20 | 4 | bit 0 encrypted, bit 1 zoned, bit 2 converting to encryption |
| volume_id | 24 | 16 | random at format |
| blocks | 40 | 8 | the volume's size in blocks |
| segments | 48 | 4 | the number of segments |
| reserved | 52 | 4 | |
| created | 56 | 8 | wall time of format, UTC nanoseconds (`i64`) |
| encryption | 64 | 1,024 | the volume key's binding ([Volumes](#volumes)) |
| reserved | 1,088 | 2,992 | |
| checksum | 4,080 | 16 | of bytes 0 to 4,079 |

## Pointers

A **pointer** is 32 bytes naming one log record:

| Field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| target | 0 | 8 | an item id when flag bit 0 is set, else the byte offset of the record in the volume |
| length | 8 | 4 | the record's payload length |
| kind | 12 | 1 | the record's kind |
| flags | 13 | 1 | bit 0: `target` is an item id |
| reserved | 14 | 2 | |
| checksum | 16 | 16 | of the record's stored bytes after its 8-byte prefix |

Every read checks the checksum of what a pointer names. Blob reads trust these checksums; the store verifies blob ids only when a blob is imported (`draft_close`), exported (`read` with `verify`) and scrubbed ([11-store.md](11-store.md#the-store-protocol)).

## Segments and log batches

The **log** is the sequence of log batches the store appends to segments. Block 0 of each segment is its header:

| Field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| magic | 0 | 8 | `NANOSSG1` |
| sequence | 8 | 8 | the order in which segments were opened |
| volume_id | 16 | 16 | |
| segment | 32 | 4 | its own number |
| next | 36 | 4 | the segment the log continues in, chosen when this one opens |
| format | 40 | 4 | |
| reserved | 44 | 4,036 | |
| checksum | 4,080 | 16 | of bytes 0 to 4,079 |

A **log batch** starts and ends at block boundaries within one segment:

| Field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| magic | 0 | 8 | `NANOSLB1` |
| batch | 8 | 8 | sequence number, consecutive across segments |
| blocks | 16 | 4 | the batch's length in blocks |
| flags | 20 | 4 | bit 0: the segment ends after this batch |
| records | 24 | 4 | the number of records |
| reserved | 28 | 4 | |
| time | 32 | 8 | wall time, UTC nanoseconds |
| checksum | 40 | 16 | of the whole batch with this field zero |

Records follow from offset 56, each starting at a multiple of 8; padding fills the last block.

## Log records

| Field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| kind | 0 | 1 | below |
| flags | 1 | 1 | bit 0: encrypted ([Volumes](#volumes)) |
| reserved | 2 | 2 | |
| length | 4 | 4 | payload length |
| key | 8 | 32 | by kind |
| payload | 40 | length | |

| Kind | Key | Payload |
| --- | --- | --- |
| 1 commit | the commit number in bytes 0–7 | wire `CommitRecord` |
| 2 value | the item id in bytes 0–7 | a record's stored bytes, ciphertext for a secret record |
| 3 piece | the piece key | up to 64 KiB of blob bytes |
| 4 index | the item id in bytes 0–7, zero for an item-table node | one 16 KiB index node |
| 5 stream | the stream id in bytes 0–7, its earliest sequence number in bytes 8–15 | wire `mem::List[StreamEntry]` |
| 6 blob | the blob id's 32-byte digest | wire `BlobMap { size: u64, pieces: mem::Bytes }`, the piece keys in order |
| 7 pad | zero | ignored |

On a volume without encryption, a piece or index record is placed so that its payload begins at a block boundary, after a pad record when needed; a space too short for a pad record moves it one block further. `CommitRecord` is `{ meta: CommitMeta, writes: mem::List[LogWrite] }`, with `LogWrite { tree: u8, dir: u64, name: mem::Bytes, entry: Option[NodeEntry] }`, `None` deleting the entry. The value, piece, stream and blob records a commit references precede its commit record, and a commit exists once its commit record is durable. A record the cleaner moves keeps its kind and key.

## Items

An **item** is a log record reached through the item table: the index nodes of trees 1 to 4 and the value records. Item ids come from the counter `next_item` and are never reused. The **item table**, tree 5, maps an item id to `ItemEntry { address: u64, length: u32, kind: u8, refcount: u32 }`, `address` being the record's byte offset in the volume. Pointers inside the item table are byte offsets, so moving an item changes only its entry. Pieces, commit records and stream runs are reached by byte offset from the one catalog entry naming each, and item-table nodes from their parent or the flush record.

## Index trees

Copy-on-write B+trees index the volume:

| Tree | Key | Leaf value |
| --- | --- | --- |
| 1 versioned, one per branch | dir id (8) ‖ name | `NodeEntry` |
| 2 unversioned, one per branch | dir id (8) ‖ name | `NodeEntry` |
| 3 machine | dir id (8) ‖ name | `NodeEntry` |
| 4 catalog | kind (1) ‖ key | [The catalog](#the-catalog) |
| 5 item table | item id (8) | `ItemEntry` |

Dir ids come from the counter `next_dir`; the top dir is 1. A node lies in tree 2 when it is below `/state` or an ephemeral dir, in tree 3 below `/journal`, else in tree 1. An index node is 16 KiB:

| Field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| magic | 0 | 4 | `NIX1` |
| tree | 4 | 1 | 1 to 5 |
| level | 5 | 1 | 0 for a leaf |
| count | 6 | 2 | its entries |
| item | 8 | 8 | its item id, 0 in tree 5 |
| flush | 16 | 8 | the index flush that wrote it |
| heap | 24 | 2 | offset of its lowest entry byte |
| reserved | 26 | 6 | |
| slots | 32 | 2·count | entry offsets in key order |

Entries fill the node from its end downwards. A leaf entry is a key length (2 bytes), the key, a value length (2) and the value, 4 KiB at most in all. An internal entry is a key length, the separating key, a pointer to the child, and the number (8) and value bytes (8) of the leaf entries below it; these subtree counts make the `count` operation ([11-store.md](11-store.md#the-store-protocol)) exact in O(log n).

`NodeEntry` is a wire record:

| Field | Type | Meaning |
| --- | --- | --- |
| `kind` | `u8` | 1 record, 2 blob, 3 dir, 4 stream |
| `version` | `u64` | |
| `subtree` | `u64` | a dir's subtree version |
| `origins` | `u32` | origin set id, 0 for none |
| `type` | `u32` | type id of a record's or stream's value |
| `declared` | `u32` | type id of the declared type when it differs from `type`, else 0 |
| `secret` | `u64` | secret id, 0 for none |
| `expires` | `i64` | expiry in UTC nanoseconds, 0 for none |
| `hash` | `Option[HashValue]` | the node's hash, once computed |
| `held` | `bool` | the node is held beyond its dir's `keep` |
| `content` | `Content` | `Inline(mem::Bytes)`, record bytes up to 1 KiB; `Value(pointer)`; `Blob { id: HashValue, size: u64 }`; `Dir { id: u64, declaration: mem::Bytes, digest: Option[HashValue] }`; `Stream { id: u64, declaration: mem::Bytes, next: u64, entries: u64, bytes: u64 }` |

## The catalog

| Kind | Key | Value |
| --- | --- | --- |
| 1 branch | name | `BranchEntry { branch: Branch, versioned: pointer, unversioned: pointer }` |
| 2 commit | number (8) | `CommitEntry { meta: CommitMeta, record: Option[pointer], tree: Option[pointer] }` |
| 3 pin | commit (8) ‖ pin number (8) | `Pin` |
| 4 request key | principal ‖ `0x00` ‖ the key's bytes | `RequestEntry { session: SessionId, commit: u64, digest: HashValue }` |
| 5 stream run | stream id (8) ‖ earliest sequence number (8) | `StreamRun { record: pointer, count: u32, bytes: u32, time: i64 }`, `time` being its newest entry's wall time |
| 6 blob | blob id (33) | `BlobEntry { size: u64, refcount: u64 }` |
| 7 blob piece | blob id (33) ‖ piece index (8) | the piece key, 32 bytes |
| 8 piece | piece key (32) | `PieceEntry { record: pointer, refcount: u64 }` |
| 9 path change | tree (1) ‖ dir id (8) ‖ name ‖ `0x00` ‖ commit (8) | `PathChange { change: u8 }`: 1 created, 2 changed, 3 deleted |
| 10 secret | secret id (8) | `SecretEntry { slot: u32 }` |
| 11 segment | segment (4) | `SegmentEntry { state: u8, live: u64, written: i64 }`: state 0 free, 1 open, 2 full, 3 freed; live bytes; time of its newest batch |
| 12 origin set | id (4) | `OriginSet { labels: mem::List[OriginLabel], refcount: u64 }` |
| 13 type | id (4) | `TypeEntry { type: TypeRef, refcount: u64 }` |
| 14 machine | empty | the machine tree's top node, a pointer |
| 15 type lookup | schema hash (33) ‖ name | the type id (4) |
| 16 origin lookup | hash of the set's wire bytes (33) | the origin set id (4) |

A commit entry keeps `record`, the commit record, while the commit is retained without a tree of its own, and `tree`, the versioned tree's top node, for a flush commit. The `Branch`, `CommitMeta` and `Pin` records are those of [11-store.md](11-store.md).

## Blob pieces

A blob of n bytes has ⌈n / 64 KiB⌉ pieces; piece i holds bytes 65,536·i to 65,536·(i + 1) − 1. Its **piece key** is the chaining value, computed without the root flag, of the subtree of the blob's BLAKE3 tree that covers chunks 64·i to 64·i + 63, or the chunks that remain for the last piece. A blob id thus follows from its piece keys alone. Pieces with equal keys, which hold equal bytes at equal offsets in any blobs, are stored once. `draft_append` writes each piece as it fills; `draft_close` writes the blob record. The blob cache holds pieces in block-aligned frames that `map` maps read-only ([05-memory.md](05-memory.md#mapped-blobs)).

## Group commit

A **log flush** writes one or more batches and makes them durable with a device cache flush, or by forced-unit-access writes ([15-devices.md](15-devices.md#block)). Log flushes are chained: one starts when the previous ends and carries every commit that arrived meanwhile, so a commit waits for two log flushes at most, and the commits of one flush become durable together. A failed device write fails every commit of its flush with `store.io`, leaves the store read-only and emits `store.device-failed`. The next mount replays that flush's batches in sequence up to the first one the device did not keep ([mounting and recovery](#mounting-and-recovery)), and the commits they hold exist from then on.

## Index flushes

An **index flush** writes the index nodes changed since the previous one and a flush record. It starts when 8 MiB of log follow the latest flush record's replay position, 30 s after the earliest log batch written since that record, under memory pressure of level 1 or more ([05-memory.md](05-memory.md#pressure)), and before power off and reboot ([08-scheduling.md](08-scheduling.md#power-off-and-reboot)); it never starts when the log has not grown. A log flush starts only when its batches leave at most 16 MiB of log after the latest durable flush record, and otherwise waits for the index flush in progress, so recovery from the latest durable flush record replays 16 MiB at most.

1. Fix the set of dirty nodes; the replay position is the next log batch.
2. Write the dirty nodes of the machine tree and of the trees of each branch whose head changed; that head becomes a **flush commit**, whose versioned tree its commit entry records. Write likewise the versioned trees of commits pinned since the previous flush.
3. Compute the root hash of each new flush commit, reusing the digests of unchanged dirs.
4. Apply the pending reference count changes, thinning, stream retention and expiries.
5. Write the catalog's and the item table's dirty nodes, then perform a log flush.
6. Write the flush record to ring slot (flush − 1) mod 64 and flush the device cache.

A device failure at any step fails the index flush with `store.io` and leaves the store read-only. A commit other than a flush commit is read through the tree of the latest flush commit before it on its branch's ancestry and the commit records written since, one flush interval at most. The flush record:

| Field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| magic | 0 | 8 | `NANOSFR1` |
| flush | 8 | 8 | sequence number, from 1 |
| batch | 16 | 8 | the log batch replay starts at |
| segment | 24 | 4 | its segment |
| block | 28 | 4 | its block in the segment |
| commit | 32 | 8 | the highest commit number covered |
| next_item | 40 | 8 | the next item id |
| next_dir | 48 | 8 | the next dir id |
| next_secret | 56 | 8 | the next secret id |
| next_stream | 64 | 8 | the next stream id |
| time | 72 | 8 | wall time, UTC nanoseconds |
| catalog | 80 | 32 | pointer to the catalog's top node |
| items | 112 | 32 | pointer to the item table's top node |
| reserved | 144 | 3,936 | |
| checksum | 4,080 | 16 | of bytes 0 to 4,079 |

## Mounting and recovery

1. Find the volume ([Placement](#placement)): `store.no-volume`.
2. Read both superblocks and take the valid one with the higher sequence; when none is valid and a copy holds the magic: `store.corrupt`.
3. Check the format version ([Format versions and upgrades](#format-versions-and-upgrades)): `store.format-unsupported`.
4. On a volume whose superblock flag bit 0 or 2 is set, have `keys` unseal the volume key ([Volumes](#volumes)): `store.locked`.
5. Read the flush ring and take the valid record with the highest flush number whose catalog and item-table nodes verify, or else the valid record before it: `store.corrupt`.
6. Replay the log from its position, following batches in sequence through segments by `next`, checking each batch's checksum and stopping before any invalid or out-of-sequence batch; register every record and apply every commit record, reading 16 MiB at most from the latest record and 32 MiB at most from the one before it. A read failure: `store.io`.
7. Free the records that no replayed commit or flushed structure references.
8. When replay applied any record, perform an index flush; the log continues in a new segment.
9. Rewrite the stale copy of each keyring block whose copies differ ([Secret records](#secret-records)), and emit `store.mounted`.

A mount failure halts the kernel with its code as the cause ([04-kernel.md](04-kernel.md#kernel-halts)). Mounting reads nothing beyond these steps: there is no full-volume check.

## Reference counting

Every item, blob entry, piece entry, origin set and type entry carries a reference count: the number of index and catalog entries of retained structures that name it.

- When the store changes a node that more than one tree references, it writes a copy: each item, blob, origin set and type the copy names gains a reference, and the shared node loses the reference of the tree that changed. A blob entry holds one reference to each of its pieces.
- Thinning a commit or deleting a branch removes the references of its trees' top nodes.
- An entry whose count reaches zero is freed, and everything it names loses one reference.

Count changes collect in memory and apply at index flushes. Space freed at flush k is reused only after flush k + 1 is durable, so a mount from the latest durable flush record or the one before it stays possible. Commit records and stream runs have one naming catalog entry and are freed with it; blob records are freed at the index flush after them.

## The cleaner

The cleaner turns segments holding little live data into free ones. A segment moves through these states, kept in its `SegmentEntry`:

| State | Event | Next | Effect |
| --- | --- | --- | --- |
| free | the log needs a segment | open | its header is written |
| open | its last batch is written | full | — |
| full | the cleaner moves its live records | freed | its space counts as free at the next index flush |
| freed | the second index flush after its freeing is durable | free | it is discarded |

1. Choose the full segment whose live fraction u is below 25% with the highest (1 − u) · age / (1 + u), age being the time since its newest batch.
2. Read its records; a record is live when its item table, piece or catalog entry points at it. A record failing its checksum emits `store.corruption`, and the segment stays full.
3. Append each live record unchanged to the log and point its entry at the copy; mark live item-table nodes dirty, so that the next index flush rewrites them.

The cleaner runs at the `background` class ([08-scheduling.md](08-scheduling.md#cpu-classes)) while the log grows and a candidate exists, and at the `normal` class when free segments fall to 4 plus 1% of all segments; an idle store never cleans. [21-performance.md](21-performance.md#requirements) bounds its write amplification. The store keeps 2 segments plus 0.5% of all segments free for itself: a write adding data that needs them fails with `store.full` as thinning states ([11-store.md](11-store.md#thinning)).

## Discards

The store discards the 8 MiB of a segment when it becomes free, and every segment at format, on devices that support discards. It never discards the head area, and an idle store issues no discard.

## Scrub

`scrub`, which the `scrub` job runs ([20-base.md](20-base.md#jobs)), verifies the volume online at the `background` class:

1. Read every live record through its pointer and check its checksum.
2. Recompute each piece's key from its bytes and offset, and each blob's id from its piece keys.
3. Check both superblocks and both keyring copies, and rewrite a damaged copy from the valid one.
4. Return `ScrubReport { records: u64, pieces: u64, blobs: u64, bytes: u64, errors: mem::List[ScrubError], duration_ms: u64 }`, with `ScrubError { address: u64, kind: u8, paths: mem::List[Path] }`, and emit `store.scrubbed` and one `store.corruption` per error.

A device read failure ends the scrub with `store.io`. Scrub repairs only the head area; the paths of a corrupt record fail to read with `store.corrupt`.

## Encryption

### Secret records

The keyring holds one slot per secret record node. Block i of the keyring exists in copy 0 and copy 1; the valid copy with the higher sequence holds its slots.

| Field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| magic | 0 | 8 | `NANOSKR1` |
| sequence | 8 | 8 | incremented at each write of the block |
| slots | 16 | 3,968 | 62 slots of 64 bytes |
| reserved | 3,984 | 96 | |
| checksum | 4,080 | 16 | of bytes 0 to 4,079 |

| Slot field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| secret | 0 | 8 | the secret id; 0 marks a free slot |
| seed | 8 | 32 | random |
| commit | 40 | 8 | the commit that created the record |
| reserved | 48 | 16 | |

- The node key of secret s is BLAKE3 in key-derivation mode, context `NaN OS store node key 1`, over the keyring master key, the slot's seed and s as 8 bytes. Only `keys` derives it ([09-authority.md](09-authority.md#keys-on-the-machine)): the store passes the seed with each value it has `keys` encrypt, and `keys` reads the seed of a record it opens with `keyring_seed` ([11-store.md](11-store.md#the-store-protocol)).
- A secret record's stored bytes, its **envelope**, are s as 8 bytes, a 24-byte random nonce, the XChaCha20-Poly1305 ciphertext of its wire bytes and the 16-byte tag; the associated data is s as 8 bytes followed by the schema hash.
- A new secret takes the lowest free slot and a seed the store draws from its `sys::Random`; the store writes both copies of its block and flushes the device before the commit holding the record becomes durable.
- `erase` checks the node's deletion through steps 1 to 8 of the [commit procedure](11-store.md#commit-procedure). Then, when a slot still holds the node's secret id, it zeroes that slot, writes the copy with the lower sequence, flushes the device, writes the other copy and flushes again. Only then does it number and append the deletion, and it answers once that commit is durable.
- A secret id that no slot holds is erased: `keyring_seed` of it and every read of a record carrying it fail with `store.erased`. A crash between the zeroing and the commit leaves the node on its branch, erased, and a repeated `erase` commits its deletion.

After an erase no operation of the machine can derive the node key; a reader of the raw device recovers it only with both an old copy of the slot that the device kept and the keyring master key. The keyring holds 61,442 slots; a secret beyond them fails with `store.full`.

### Volumes

Volume encryption, which the owner chooses at [claim](19-oversight.md#claim) on a machine with `tpm`, encrypts all but the superblocks, segment headers, batch headers and record prefixes under the 256-bit volume key.

- The superblock's `encryption` field holds the scheme (4 bytes, 1 for XChaCha20-Poly1305), the length (2 bytes) and the bytes of the volume key sealed to the TPM with no PCR selected ([15-devices.md](15-devices.md#tpm)). At mount `store` passes this blob to `keys` through its `KeyUse(volume)`, and `keys` unseals it through the TPM ([09-authority.md](09-authority.md#keys-on-the-machine)); a refusal fails the mount with `store.locked`. Code booted on the machine in NaN OS's place unseals that blob as `keys` does.
- A log record with flag bit 0 stores, after its 8-byte prefix, a 24-byte random nonce, the XChaCha20-Poly1305 ciphertext of its key and payload, and the tag; the associated data is the prefix. A head-area block other than a superblock stores, after its 16-byte header, a nonce, the ciphertext of bytes 16 to 4,039 of its plain form, and the tag; the associated data is the header.
- At claim `keys` generates and seals the volume key; the store writes the sealed blob, sets flag bit 2 and rewrites the head area encrypted. From then on it writes records encrypted, and the cleaner rewrites every segment holding plain records; when none remains, the store sets bit 0 and clears bit 2.

## Format versions and upgrades

The superblock's `format` is the volume's format version; this specification defines format 1. A kernel whose generation manifest declares `store_format` v ([03-boot.md](03-boot.md#generations-and-manifests)) mounts volumes of formats v and v − 1 and writes structures of the volume's own format; another format fails with `store.format-unsupported`. It upgrades a volume of format v − 1 only after its generation passes the [health gate](03-boot.md#health-gate), and only when every generation in the boot record's slots and its recovery generation declares `store_format` v or more: it rewrites both superblocks with format v, writes format v from then on, lets the cleaner rewrite older structures, and emits `store.format-upgraded`.

## Events

| Kind | Payload | Section |
| --- | --- | --- |
| `store.mounted` | `{ placement: text::String, format: u32, replayed: u64, duration_ms: u32 }` | [Mounting and recovery](#mounting-and-recovery) |
| `store.corruption` | `ScrubError` | [Conventions](#conventions) |
| `store.device-failed` | `{ code: text::String }` | [Group commit](#group-commit) |
| `store.scrubbed` | `ScrubReport` | [Scrub](#scrub) |
| `store.format-upgraded` | `{ from: u32, to: u32 }` | [Format versions and upgrades](#format-versions-and-upgrades) |
