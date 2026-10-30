# 0010. Boot by signed generations with an anti-rollback floor

Status: accepted

## Context

A kernel update must never leave a machine unbootable, and an attacker with disk access must not boot an old, vulnerable system. The boot record that counts tries is written by a loader holding no signing key, so it cannot be authenticated. NaN's UEFI programs reach the firmware only through `write_console` and `read_file`, and cannot hand control to another image.

## Decision

- A separate NaN loader is the only code the firmware starts, and it is never live-updated.
- A **generation** is a signed, immutable manifest naming a kernel image and a genesis record.
- The mutable boot record (two slots, a recovery entry, tries, states) only orders choices among manifests that verify. No security decision rests on it.
- An **anti-rollback floor** is kept in a TPM NV counter, or in a boot-services-only UEFI variable that only a loader the firmware ran can write. The loader refuses generations below it.
- A trial generation runs on its own store branch until its health gate marks it good. A failed trial leaves `main` as it was.
- A halt reboots only through a hardware watchdog, and the loader's try counters then fall back.
- A production loader boots only verified generations. A development loader, built for the sandbox, also boots `/kernel.elf` unverified in development mode.

Alternatives rejected:

- **Merging the loader and the kernel into one UEFI application:** nothing would choose a slot before the kernel runs.
- **Signing the mutable boot record:** the loader has no key.
- **A recovery mode chosen by a file on the boot volume:** anyone with disk access could force it.
- **Live update of the core:** complex, where a reboot that loses no committed state is cheap.

## Consequences

- Without a TPM and without Secure Boot, a physical attacker can still replace the loader. The machine reports `rollback: unprotected`.
- The loader needs firmware writes, variables, TCG2, RNG, configuration tables, console input and a hand-off to a loaded image ([N14](../spec/22-nan.md#n14)), and placement clear of firmware memory ([N31](../spec/22-nan.md#n31)).
- The sandbox cannot reboot within a run, so boot counting and fallback are unobservable there ([N22](../spec/22-nan.md#n22)).
