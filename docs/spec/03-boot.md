# Boot

This chapter defines the boot volume, generations and manifests, the boot record, the floor and owner key storage, the loader, measurement, entropy, the handoff record, the modes, the development loader, direct boot, the health gate, recovery, reboot after a halt, generation operations, loader replacement, and owner key requests, claims included.

## Boot volume

The **boot volume** is the FAT EFI system partition holding the loader, the generations and the boot record. The loader reaches it through firmware services (requires [N14](22-nan.md#n14)), the kernel through `fat` ([15-devices](15-devices.md#the-fat-component)) on the partition the [handoff record](#handoff-record) names.

| Path | Holds | Written by |
| --- | --- | --- |
| `/EFI/BOOT/BOOTX64.EFI` (`BOOTAA64.EFI` on AArch64) | the loader | the installer; the loader, at a [loader replacement](#loader-replacement) |
| `/EFI/nanos/loader.prev` | the loader the last replacement replaced | the loader |
| `/EFI/nanos/loader.new` | a loader waiting to replace the current one | `boot` |
| `/nanos/boot.rec`, `/nanos/boot.bak` | copies 0 and 1 of the [boot record](#boot-record) | the loader, `boot` |
| `/nanos/gen/<n>/` | generation n, n in decimal: `manifest`, its signed manifest; `kernel.elf`, its kernel image; `genesis`, its genesis record | the installer, `boot` |
| `/nanos/store.img` | the store, on a machine without a store partition ([12-store-format](12-store-format.md#placement)) | `devices`, through `fat` ([15-devices](15-devices.md#volumes-and-roles)) |
| `/kernel.elf` | the kernel image of a development boot | `nan run` |

An installer writes the loader, one generation, and a boot record holding that generation in slot A as `good` and as recovery generation. At run time only the [base instance](20-base.md#boot) `boot` changes the boot volume, `/nanos/store.img` excepted; it mirrors the boot record in `/sys/boot` ([13-tree](13-tree.md#sys)).

## Generations and manifests

A **generation** is a bootable system: a signed manifest naming a kernel image and a genesis record. Its manifest is a `Manifest` record:

| Field | Type | Meaning |
| --- | --- | --- |
| `generation` | `u64` | its number, at least 1, the n of its directory |
| `kernel` | `HashValue` | the blob id of its `kernel.elf` |
| `genesis` | `HashValue` | the blob id of its `genesis` |
| `min_loader` | `u32` | the lowest loader version that boots it |
| `store_format` | `u32` | the newest store format version its kernel reads and writes, besides the version before it ([12-store-format](12-store-format.md#format-versions-and-upgrades)) |
| `signer` | `Principal` | `owner` or a `vendor.<name>` principal |

- The file `manifest` holds the `mem::wire` encoding of a `SignedManifest` record: `manifest`, the `Manifest`, and `signatures`, a `mem::Bounded[Signature, 2]` of the signer's signatures over the manifest hash, which is the hash of kind `manifest` of the `Manifest`'s `mem::wire` bytes ([02-identifiers](02-identifiers.md#hashes)).
- A manifest verifies when one of its signatures verifies with a key of its signer: for `vendor.<name>`, a key the loader's build holds under that principal; for `owner`, the [stored owner key](#floor-and-owner-key-storage).
- A manifest never changes, except its signatures, which `boot` updates when the owner key changes ([Owner key requests](#owner-key-requests)).
- The genesis file holds the `mem::wire` encoding of the genesis record ([13-tree](13-tree.md#genesis)).

## Boot record

The **boot record** orders the loader's choice among generations. Each copy is a 512-byte layout:

| Field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| magic | 0 | 8 | ASCII `NANOSBR1` |
| format | 8 | 4 | 1 |
| reserved | 12 | 4 | 0 |
| sequence | 16 | 8 | incremented by each write |
| boot counter | 24 | 8 | incremented at each boot |
| slot A | 32 | 16 | a slot entry |
| slot B | 48 | 16 | a slot entry |
| recovery generation | 64 | 8 | the generation recovery mode boots, 0 for none |
| floor request | 72 | 8 | a generation to raise the floor to, 0 for none |
| next once | 80 | 1 | the next boot's choice alone: 0 none, 1 slot A, 2 slot B, 3 recovery |
| owner request kind | 81 | 1 | 0 none, else the kind of the [owner request](#owner-key-requests) |
| reserved | 82 | 6 | 0 |
| owner request key | 88 | 32 | the requested owner key |
| owner request recovery | 120 | 32 | the requested recovery digest |
| owner request proof | 152 | 64 | the request's proof |
| loader request | 216 | 33 | the blob id of `/EFI/nanos/loader.new`, zeros for none |
| reserved | 249 | 3 | 0 |
| loader request version | 252 | 4 | that loader's version |
| loader request signature | 256 | 64 | the owner's signature of the [loader request](#loader-replacement) |
| reserved | 320 | 176 | 0 |
| checksum | 496 | 16 | XXH3-128 with seed 0 of bytes 0 to 495, its low 64 bits at 496 |

A slot entry is 16 bytes:

| Field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| generation | 0 | 8 | 0 for none |
| state | 8 | 1 | 0 `empty`, 1 `trial`, 2 `good`, 3 `bad` |
| tries | 9 | 1 | the trial boots left, at most 3 |
| reserved | 10 | 6 | 0 |

- A copy is valid when it holds the magic, format 1, a matching checksum, zero reserved bytes and defined enumerated values. The record is the valid copy of higher sequence, copy 0 on a tie; with neither valid there is no record.
- A write sets the sequence to the record's plus one and goes to the copy not holding the record, copy 0 when there is no record. The loader writes it in step 6 only.
- The loader writes the boot counter and the tries, sets the slots it rejects `bad`, and clears `next once` and the requests it handles. `boot` writes every other field ([20-base](20-base.md#boot)) and, at its start when the handoff flag `record written` is clear, the handoff record's boot counter.
- The owner and loader requests carry proofs the loader checks; every other field only orders the choice among generations that verify and pass the floor.

| State | Event | Next | Effect |
| --- | --- | --- | --- |
| `empty`, `good`, `bad` | `boot` installs a generation in it | `trial` | tries 3; `next once` names it |
| `trial` | the loader boots it | `trial` | tries minus 1 |
| `trial` | the loader finds no tries left | `bad` | none |
| `trial`, `good` | its files fail verification | `bad` | none |
| `trial` | the health gate passes | `good` | `main` takes `trial/<n>` |
| `good` | `boot` rolls back from it | `bad` | reboot |
| `trial`, `good`, `bad` | `boot` retires its generation | `empty` | its directory and its trial branch are deleted |

## Floor and owner key storage

The **floor** is the lowest generation the loader boots. The owner key is the Ed25519 public key of `owner` ([09-authority](09-authority.md#principals)); the owner record stores it, followed by the BLAKE3-256 digest of the owner's recovery code.

| Value | TPM NV index, on a machine with `tpm` | UEFI variable |
| --- | --- | --- |
| floor | `0x0180_4E00`: the floor, a `u64` | `NanosFloor`: the same 8 bytes |
| owner record | `0x0180_4E01`: 64 bytes | `NanosOwner`: the same 64 bytes |

- The variables have the vendor GUID `032e8f8b-8377-431e-9bae-10c416c21bef` and the attributes non-volatile and boot-services access only.
- The NV indices are ordinary indices of the owner range, with an empty authorization value and the attributes `TPMA_NV_AUTHREAD`, `TPMA_NV_AUTHWRITE`, `TPMA_NV_WRITE_STCLEAR` and `TPMA_NV_NO_DA`. The loader defines them when absent and write-locks both (`TPM2_NV_WriteLock`) before `exit_services`, for the rest of the boot.
- The floor is the greater of the stored values, 0 when none is stored. The owner record is the variable's when it exists, else the index's. With no owner record stored, the machine boots unclaimed ([19-oversight](19-oversight.md#claim)).
- The loader writes the variable, then the index, and only ever raises the floor. At each boot, before write-locking the indices, it writes the floor and the owner record into each of them not holding it.
- A machine on which the loader stores the floor in neither, flags 4 and 5 of the [handoff record](#handoff-record) being clear, has no floor, reported in `/live/security` as `rollback: unprotected` ([13-tree](13-tree.md#live)), and can keep no owner record: a claim fails with `policy.unclaimable` ([claim](19-oversight.md#claim)).

## Loader

The **loader** is the UEFI program that chooses, verifies, measures and starts a generation. Its build fixes its version, a `u32`, the vendor keys it accepts, each a `vendor.<name>` principal with an Ed25519 public key, and whether it is a [development loader](#development-loader). It uses firmware services for files, variables, the TPM, the RNG, the memory map, the device path of its image and console input (requires [N14](22-nan.md#n14)).

Each failure prints the line `loader: <code> generation <n>` on the firmware console, n being 0 when no generation applies. A boot that proceeds prints `loader: generation <n> slot <slot> mode <mode>` before step 9. Its steps:

1. **Read the record.** It reads both copies, each `Untrusted`, through a validator. With no record it prints `boot.record-invalid` and continues with every slot `empty`. A pending `r` on the console input, read without waiting, requests recovery.
2. **Apply the requests**: the [owner request](#owner-key-requests) and the [loader request](#loader-replacement). Their outcomes enter the handoff record's flags; neither stops the boot.
3. **List the candidates.** Each generation appears once, and slots `empty` or `bad` never. The recovery list holds the recovery generation, then the generations of the `good` slots, then those of the other directories of `/nanos/gen`, each in descending generation order, all in `recovery` mode. With recovery requested, the candidates are the recovery list; otherwise, in order:
    1. the target of `next once`: a `trial` slot in `trial` mode, a `good` slot in `normal` mode, or for 3 the recovery list;
    2. the `trial` slot in `trial` mode; with no tries left the loader sets it `bad` and prints `boot.tries-exhausted`;
    3. the `good` slots in descending generation order, in `normal` mode;
    4. the recovery list.

    A development loader then appends `/kernel.elf` in `dev` mode.
4. **Verify the candidates** in order; the earliest that passes boots:
    1. read `/nanos/gen/<n>/manifest` (`boot.manifest-unreadable`) and decode it (`boot.manifest-malformed`);
    2. find a key of its signer (`boot.signer-unknown`) with which one of its signatures verifies (`boot.signature-invalid`);
    3. check that its generation is n and, for a slot, the slot's (`boot.generation-mismatch`), at least the floor (`boot.below-floor`), and that its `min_loader` is at most the loader's version (`boot.loader-too-old`);
    4. read `kernel.elf` and `genesis` (`boot.file-unreadable`) into the loader's boot heap (`boot.too-large`) and compare their blob ids with the manifest's (`boot.hash-mismatch`);
    5. validate the kernel image: an ELF64 executable for the architecture, with disjoint loadable segments inside the file and, on x86_64, a `XEN_ELFNOTE_PHYS32_ENTRY` note ([NaN 27 §Images](https://github.com/nan-language/nan/blob/main/docs/spec/27-program-model.md#images)) (`boot.image-malformed`), whose physical ranges lie in usable memory of the current memory map (`boot.placement`, requires [N31](22-nan.md#n31)).

    The `dev` candidate takes only sub-steps 4, for `/kernel.elf` alone, and 5. A failing candidate enters the handoff record's failures, up to four, and its slot becomes `bad` unless its code is `boot.manifest-unreadable`, `boot.file-unreadable` or `boot.loader-too-old`. With no candidate left, the loader prints `boot.none-bootable` and stops.
5. **Raise the floor.** In `normal` mode, the loader clears any floor request; a request above the floor, at most the booting generation and equal to the generation of a `good` slot becomes the [floor](#floor-and-owner-key-storage), and any other leaves the floor unchanged.
6. **Write the record**: the boot counter plus 1, `next once` cleared, the tries of the booting `trial` slot minus 1, the `bad` states of steps 3 and 4, and the handled requests cleared. A failed write prints `boot.record-unwritable`; the loader then abandons a `trial` candidate for the next candidate not in `trial` mode, from step 4, and boots any other with the handoff flag `record written` clear.
7. **Measure** the candidate on a machine with `tpm` ([Measurement](#measurement)).
8. **Draw entropy** ([Entropy](#entropy)) and copy the TPM event log into the loader's memory.
9. **Hand off.** It calls `exit_services`, copies the kernel's loadable segments to their physical addresses, writes the handoff block and enters the kernel ([Handoff record](#handoff-record)). A segment outside usable memory of the final memory map instead prints `loader: boot.placement generation <n>` on the descriptor's serial port, and the loader stops.

## Measurement

On a machine with `tpm`, the loader extends PCRs with `EFI_TCG2_PROTOCOL.HashLogExtendEvent`, which extends every active bank, logging events of type `EV_IPL` (`0x0000_000D`) whose event data are the measured bytes:

| Order | PCR | Measured bytes |
| --- | --- | --- |
| 1 | 8 | the manifest hash, 33 bytes |
| 2 | 9 | the blob id of the kernel image, 33 bytes |
| 3 | 9 | the blob id of the genesis record, 33 bytes |
| 4 | 8 | the UTF-8 text `nanos.mode`, a space and the mode |

- A `dev` boot measures events 2 and 4 only.
- After its last measurement the loader copies the event log, in the TCG PC Client crypto-agile format (`EFI_TCG2_EVENT_LOG_FORMAT_TCG_2`), and passes the address of the `EFI_TCG2_FINAL_EVENTS_TABLE` configuration table, which holds the events logged after the copy.

## Entropy

- The loader draws 64 bytes for the handoff record from `EFI_RNG_PROTOCOL`, with its default algorithm, when the firmware provides it; otherwise the field is zero and its flag clear.
- The kernel's random source is seeded once it has mixed at least 256 bits from the handoff record, RDSEED on a CPU that has it (requires [N11](22-nan.md#n11)), and the entropy devices ([15-devices](15-devices.md#device-classes)). The link opens no transport and `keys` generates no key before that. The kernel overwrites the handoff entropy once mixed.

## Handoff record

The **handoff record** is what the loader passes to the kernel: a 512-byte layout.

| Field | Offset | Size | Meaning |
| --- | --- | --- | --- |
| magic | 0 | 8 | ASCII `NANOSHO1` |
| version | 8 | 4 | 1 |
| size | 12 | 4 | 512 |
| generation | 16 | 8 | the booted generation, 0 in `dev` mode |
| boot counter | 24 | 8 | the boot record's boot counter plus 1 |
| floor | 32 | 8 | as step 5 left it |
| loader version | 40 | 4 | the loader's version |
| flags | 44 | 4 | the flags below |
| volume partition | 48 | 4 | the boot volume's partition number |
| mode | 52 | 1 | 0 `normal`, 1 `trial`, 2 `recovery`, 3 `dev` |
| slot | 53 | 1 | the slot booted: 1 A, 2 B, else 0, as in `recovery` and `dev` modes |
| tries | 54 | 1 | that slot's tries after step 6 |
| conditions | 55 | 1 | bit 0 `tpm`, bit 1 `secure-boot`, other bits 0 |
| failure count | 56 | 1 | the failures entries in use, 0 to 4 |
| volume scheme | 57 | 1 | the boot volume's partitioning: 0 unknown, 1 MBR, 2 GPT |
| reserved | 58 | 6 | 0 |
| volume signature | 64 | 16 | the GPT unique partition GUID, or the MBR disk signature in bytes 0 to 3 |
| owner key | 80 | 32 | the stored owner key, zeros when unclaimed |
| recovery digest | 112 | 32 | the stored digest of the recovery code |
| manifest hash | 144 | 33 | zeros in `dev` mode |
| kernel hash | 177 | 33 | the kernel image's blob id |
| genesis hash | 210 | 33 | the genesis record's blob id, zeros when the image holds it |
| loader hash | 243 | 33 | the blob id of the loader's file |
| reserved | 276 | 4 | 0 |
| event log address | 280 | 8 | 0 without `tpm` |
| event log size | 288 | 8 | in bytes |
| final events address | 296 | 8 | the address of `EFI_TCG2_FINAL_EVENTS_TABLE`, 0 without it |
| entropy | 304 | 64 | from the RNG, or zeros |
| failures | 368 | 128 | 4 entries of 32 bytes: a generation (`u64`), then a code in ASCII, zero-padded |
| entry counter | 496 | 8 | the time-stamp counter, `CNTVCT_EL0` on AArch64, at the loader's entry (requires [N11](22-nan.md#n11), [N14](22-nan.md#n14)) |
| reserved | 504 | 8 | 0 |

The flags are, by bit: 0, a development loader; 1, `record written`, step 6 wrote the record; 2, entropy present; 3, recovery requested on the console; 4 and 5, the floor stored in the TPM and in a UEFI variable; 6, the floor raised by step 5; 7 and 8, the owner request applied or refused; 9 and 10, the loader request applied or refused; 11, a UEFI boot entry for the previous loader.

The **handoff block** is a range of pages below 4 GiB, in usable memory outside the kernel's segments, holding the start information, the handoff record, the genesis record and the event log copy.

- On x86_64 the loader enters the kernel at its PVH 32-bit entry (requires [N14](22-nan.md#n14)) with `ebx` holding the address of a PVH start information structure, version 1, in the handoff block. Its module list holds module 0, the handoff record, and module 1, the genesis record, absent when the image holds it; `rsdp_paddr` is the ACPI RSDP of the EFI configuration table and `cmdline_paddr` is 0. Its memory map gives each range of the final UEFI memory map a type: 1 for `EfiConventionalMemory`, `EfiBootServicesCode`, `EfiBootServicesData`, `EfiLoaderCode` and `EfiLoaderData`; 3 for `EfiACPIReclaimMemory`; 4 for `EfiACPIMemoryNVS`; 5 for `EfiUnusableMemory`; 7 for `EfiPersistentMemory`; and 2 for the handoff block and every other type.
- On AArch64 the loader enters the image at EL1 with the MMU off and `x0` holding the address of a flattened device tree in the handoff block: the firmware's device tree when the EFI configuration table holds one, else a tree holding the memory of the final map and `/chosen/nanos,acpi-rsdp`. It adds `/chosen/nanos,handoff` and `/chosen/nanos,genesis`, each an address and a size in two 64-bit cells, and lists the handoff block in the memory reservation block (requires [N30](22-nan.md#n30)).
- At start the kernel emits `boot.started`, then one `boot.candidate-failed` per failure entry.

## Modes

| Mode | Booted from | The kernel |
| --- | --- | --- |
| `normal` | a `good` slot | runs on the branch `main` |
| `trial` | a `trial` slot | runs on the branch `trial/<n>` under the [health gate](#health-gate) |
| `recovery` | the recovery list | runs as [Recovery](#recovery) states |
| `dev` | `/kernel.elf` | treats the console session as the owner ([18-link](18-link.md#connections-and-sessions)); has no slot, floor change or health gate |

## Development loader

- A development loader is built with the `loader` package's `DEVELOPMENT` constant set to true. Its `/kernel.elf` candidate boots unverified, in `dev` mode, as generation 0. A production loader never boots a kernel image without a verifying manifest.
- A development boot has no manifest. Its kernel commits the genesis record its image holds ([13-tree](13-tree.md#genesis)).
- Flag 0 of the handoff record and `/live/security` mark a development loader. A vendor must not sign a development loader for secure boot.

## Direct boot

A program other than the loader may enter a kernel image with the start information of [Handoff record](#handoff-record): a hypervisor through PVH, or an AArch64 firmware with a device tree. That program supplies the handoff record and the genesis record, and the kernel trusts it as it trusts the loader. Its handoff record holds slot 0, mode `normal` or `dev`, and volume scheme 0, so no boot volume exists: the health gate does not apply and the operations of `boot` fail with `boot.no-boot-volume`.

## Health gate

The **health gate** decides when a `trial` generation becomes `good`.

- It passes once these hold together, without interruption, for 30 s by default (`HealthGate.window_s`): the store is mounted ([11-store](11-store.md#transactions)), every base instance is `converged` ([13-tree](13-tree.md#desired-and-observed-state)), and the link accepts requests on a transport ([18-link](18-link.md#transports)).
- When it passes, `boot` sets the slot `good`, then has the store take `trial/<n>` into `main` ([trial branches](11-store.md#trial-branches)), and the kernel emits `boot.generation-good`.
- A `normal` boot whose generation's trial branch still exists runs on it, and `boot` has the store take it as when the gate passes.
- When the gate has not passed 300 s by default (`HealthGate.deadline_s`) after the kernel's start, the kernel emits `boot.gate-timeout` and reboots through `power` ([08-scheduling](08-scheduling.md#power-off-and-reboot)); when that reboot fails, the kernel halts (`fault.power-failed`).
- A halt, reset or reboot before the gate passes consumes the boot's try. When the tries are exhausted, the next boot runs `main`, `trial/<n>` being kept.

`/sys/config/boot` holds the `BootConfig` record of `boot`, whose `watchdog_s` is the [watchdog](04-kernel.md#watchdog)'s timeout:

```nan
pub closed struct HealthGate derives Copy, Eq, Render, Encode {
    pub window_s: u32
    pub deadline_s: u32
} HealthGate

pub closed struct BootConfig derives Copy, Eq, Render, Encode {
    pub health: HealthGate
    pub watchdog_s: u32
} BootConfig
```

## Recovery

- Recovery mode boots when recovery is requested on the console, when `next once` is 3, when no record exists, and when no slot candidate verifies.
- In recovery mode the kernel starts the base instances only, loads no agent bundle and applies no health gate. `boot` accepts operations from owner sessions only, and the console accepts the replacement of a lost owner key with the recovery code ([19-oversight](19-oversight.md#recovery)).
- In recovery mode the store accepts only these versioned writes; every other fails with `store.read-only` ([commit procedure](11-store.md#commit-procedure)):
    - an owner session's revert;
    - the commits made as `system` ([commits](11-store.md#commits));
    - the writes of `/sys/boot` by `boot`.
- `boot` keeps the recovery generation at or above the floor and never deletes its directory.

## Reboot after a halt

- After a kernel halt ([NaN 24 §Kernel halts](https://github.com/nan-language/nan/blob/main/docs/spec/24-kernel.md#kernel-halts)), on a machine with `watchdog`, the [watchdog](04-kernel.md#watchdog) resets the machine within its timeout, and the loader counts the boot as the [health gate](#health-gate) states.
- Without `watchdog`, a halted machine stays stopped unless reset by hand.

## Generation operations

These operations of `boot` ([20-base](20-base.md#boot)) change the boot volume.

- `install`, for the plan step `generation` ([14-changes](14-changes.md#steps)):
    1. Verify the manifest as step 4 of the loader does, against the handoff record's floor and loader version, and check that the store's format version is the manifest's `store_format` or the one before it (`boot.store-format`). An install in `trial` mode fails with `boot.trial-pending`.
    2. Write `manifest`, `kernel.elf` and `genesis` to `/nanos/gen/<n>/`, reading each back and comparing its blob id (`boot.write-failed`, `boot.volume-full`).
    3. Write the record: the idle slot becomes `trial` with generation n and 3 tries, and `next once` names it. The idle slot is the slot not holding the running generation; in `recovery` and `dev` modes it is an `empty` or `bad` slot, else the one with the lower generation. The kernel emits `boot.generation-installed`.
    4. The [generation switch](10-code.md#generation-switch) reboots.
- `rollback` sets the running slot `bad` and reboots, when the other slot is `good` with a generation at or above the floor (`boot.no-fallback`).
- `recovery` sets `next once` to 3 and reboots.
- `retire` to generation g writes the floor request g, when g is at most the running generation and the recovery generation (`boot.floor-too-high`). Once a boot reports the floor raised, `boot` empties the slots holding generations below the floor, deletes their directories and has the store delete `trial/<k>` for every k below the floor, and the kernel emits `boot.retired`; a request the loader cleared without raising the floor leaves `/sys/boot` showing the unchanged floor, and `retire` may be repeated.
- `set_recovery` makes generation g the recovery generation when g is the generation of a `good` slot at or above the floor (`boot.not-good`).

## Loader replacement

Replacing the loader is an `irreversible{actuate}` operation of `boot` ([14-changes](14-changes.md#reversibility)).

1. `replace_loader` writes the new loader to `/EFI/nanos/loader.new`, reads it back, and writes the loader request: the new loader's blob id, its version, and the owner's signature of kind `request` ([02-identifiers](02-identifiers.md#hashes)) over the `mem::wire` encoding of a `LoaderRequest` record of fields `loader`, that blob id (`HashValue`), and `version`, that version (`u32`), having verified that signature ([`auth.bad-signature`](09-authority.md#codes)). The plan then reboots.
2. In step 2 of the next boot, the loader applies the request when the signature verifies with the owner key, the version exceeds its own and `loader.new` has the requested blob id: running from `/EFI/BOOT/BOOTX64.EFI`, it copies that file to `/EFI/nanos/loader.prev`; it sets the UEFI boot entries `NaN OS` for `\EFI\BOOT\BOOTX64.EFI` and `NaN OS previous loader` for `\EFI\nanos\loader.prev` at the head of `BootOrder`, in that order; it replaces `/EFI/BOOT/BOOTX64.EFI` with `loader.new`, reading it back, and deletes `loader.new`. Otherwise it refuses the request. The kernel emits `boot.loader-request`.
3. The new loader starts at the boot after. When the firmware cannot start it, or refuses it on a machine with `secure-boot`, the firmware starts the previous loader through its boot entry.

## Claim at boot

A claim, like each replacement of the owner key, is an owner request, applied at the next boot.

### Owner key requests

An owner request asks the loader to store a new owner record:

| Kind | Applies when | Proof |
| --- | --- | --- |
| 1, claim | no owner record is stored | 64 zero bytes |
| 2, replacement by signature | the proof verifies with the stored owner key | the stored owner's signature of kind `request` over the `mem::wire` encoding of an `OwnerKeyRequest` record of fields `current`, `key` and `recovery`: the stored key, the requested key and the requested recovery digest, each an `Array[u8, 32]` |
| 3, replacement by recovery code | the BLAKE3-256 digest of the proof's leading 16 bytes is the stored recovery digest | the recovery code (16 bytes), then 48 zero bytes |

- Applying a request stores its key and recovery digest as the owner record; a request whose key is the stored key counts as applied, and any other that does not apply is refused. The kernel emits `boot.owner-request`.
- `boot` writes kind 1 through `claim`, with the digest of the recovery code the [claim](19-oversight.md#claim) draws; kind 2 through `replace_owner`, after verifying the signature ([`auth.bad-signature`](09-authority.md#codes)); and kind 3 through `recover`, in recovery mode, after checking the code against the handoff record's recovery digest ([`policy.recovery-code-invalid`](19-oversight.md#codes)), with the digest of the new recovery code ([recovery](19-oversight.md#recovery)).
- With a kind 2 or 3 request, `boot` adds a signature by the requested key to the manifest of every generation signed by `owner` that the boot record names, replacing any signature by neither key, so each verifies under either key, reads each manifest back, and only then writes the request. A request lacking one of these signatures fails with `boot.resign-required`, and a manifest write that fails or reads back different with `boot.write-failed`; neither writes the request.
- The handoff record's owner key is the key of `owner` for the boot ([09-authority](09-authority.md#principals)).

## Events

| Kind | Payload | Section |
| --- | --- | --- |
| `boot.started` | mode, generation, slot, boot counter, flags | [Handoff record](#handoff-record) |
| `boot.candidate-failed` | generation, code | [Loader](#loader) |
| `boot.generation-good` | generation, slot | [Health gate](#health-gate) |
| `boot.gate-timeout` | generation | [Health gate](#health-gate) |
| `boot.owner-request` | kind, applied | [Owner key requests](#owner-key-requests) |
| `boot.loader-request` | blob id, applied | [Loader replacement](#loader-replacement) |
| `boot.generation-installed` | generation, slot | [Generation operations](#generation-operations) |
| `boot.retired` | floor, generations deleted | [Generation operations](#generation-operations) |

## Codes

| Code | Meaning | Section |
| --- | --- | --- |
| `boot.record-invalid` | no copy of the boot record is valid | [Loader](#loader) |
| `boot.tries-exhausted` | a `trial` slot has no tries left | [Loader](#loader) |
| `boot.manifest-unreadable` | a manifest is missing or unreadable | [Loader](#loader) |
| `boot.manifest-malformed` | a manifest does not decode | [Loader](#loader) |
| `boot.signer-unknown` | no key of the signer is known | [Loader](#loader) |
| `boot.signature-invalid` | no signature verifies | [Loader](#loader) |
| `boot.generation-mismatch` | a manifest's generation differs from its directory's or slot's | [Loader](#loader) |
| `boot.below-floor` | a generation is below the floor | [Loader](#loader) |
| `boot.loader-too-old` | `min_loader` exceeds the loader's version | [Loader](#loader) |
| `boot.file-unreadable` | a kernel image or genesis record is missing or unreadable | [Loader](#loader) |
| `boot.too-large` | a file exceeds what the loader's boot heap holds | [Loader](#loader) |
| `boot.hash-mismatch` | a file's blob id differs from its manifest's | [Loader](#loader) |
| `boot.image-malformed` | a kernel image is no ELF64 executable the loader can start | [Loader](#loader) |
| `boot.placement` | a loadable segment lies outside usable memory | [Loader](#loader) |
| `boot.record-unwritable` | a boot record write failed | [Loader](#loader) |
| `boot.none-bootable` | no candidate verifies | [Loader](#loader) |
| `boot.no-boot-volume` | the kernel was entered by direct boot | [Direct boot](#direct-boot) |
| `boot.store-format` | a generation cannot read the store's format version | [Generation operations](#generation-operations) |
| `boot.trial-pending` | an install while a `trial` generation runs | [Generation operations](#generation-operations) |
| `boot.write-failed` | a boot volume write failed or read back different | [Generation operations](#generation-operations) |
| `boot.volume-full` | the boot volume lacks space | [Generation operations](#generation-operations) |
| `boot.no-fallback` | no other `good` slot is at or above the floor | [Generation operations](#generation-operations) |
| `boot.floor-too-high` | a retirement above the running or recovery generation | [Generation operations](#generation-operations) |
| `boot.not-good` | the generation is in no `good` slot at or above the floor | [Generation operations](#generation-operations) |
| `boot.resign-required` | a key replacement lacks the new key's manifest signatures | [Owner key requests](#owner-key-requests) |
