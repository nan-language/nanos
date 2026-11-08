# 0030. The core's random source

Status: accepted

## Context

Chapter 03 has the kernel mix into its random source the entropy of the handoff record, overwritten once mixed, RDSEED and the entropy devices, which chapter 15 has `devices` reseed it from; chapter 09's `Random` grant mints a `sys::Random` drawing from it. NaN's std holds no generator or cipher, and ident holds BLAKE3. `sys::RandomService.fill` cannot wait and runs within 4 KiB of stack, where BLAKE3's subtree stack alone takes 1.7 KiB, and NaN proves neither BLAKE3's termination nor its absence of traps, which a body holding a spinlock needs. A module's memory, where the handoff record lies, cannot be written (N34), and RDSEED is not yet an intrinsic (N11).

## Decision

- **The source.** A 32-byte key and a count of the numbers its draws have taken. Mixing replaces the key by the BLAKE3 digest of the key, a tag and the input, 991 bytes at a time, so that key, tag and input fill one chunk; a draw hashes each 32-byte block from the key under a number no other draw takes, then replaces the key by a digest of itself under one more, so that no later key gives an earlier draw. Each derivation hashes a single chunk, without BLAKE3's subtree stack.
- **Sharing.** A spinlock guards the key and the count alone: a draw takes its numbers and the key under it and hashes outside it, and a change replaces the key only if it is still the one it read, a mix trying again on a newer key and a draw leaving a newer one in place.
- **Inputs.** The kernel mixes the handoff record's entropy at its start and zeroes its own copy. The keeper mixes what `devices` sends through the root's `Mix` order: 64 bytes from each entropy device whenever an entropy driver binds, and after each suspension order to an entropy driver.
- **Draws.** The kernel mints a `sys::Random` over the source for the runner, which gives it to each world whose specification holds a `Random` grant.

Alternatives rejected:

- **Hashing under the spinlock:** BLAKE3 does not check as `notrap` and `terminates`.
- **A task serving draws:** `fill` cannot wait for it.
- **Mixing by XOR of digests:** a homemade combination where a key that is still the one read can simply be replaced.

## Consequences

- Before its first mix the source derives from a zero key, and nothing yet tells whether it is seeded.
- The handoff record's entropy stays in its module's memory until NaN lets the kernel overwrite it.
- Each reseed mints a port to the driver, held until the driver is deleted.
