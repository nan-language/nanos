# 0002. Sandbox instances

Status: accepted

## Context

The sandbox boots NaN OS under QEMU through `nan run`, which keeps a program's images, and the FAT boot volume it remakes at each launch, in the `.nan` directory of the workspace root. Two runs from one workspace would overwrite each other's images and boot volume, yet sessions, the user's included, run instances side by side, sometimes from one checkout.

## Decision

- An instance is a copy of the working tree in `.sandbox/<instance>/`, a NaN workspace of its own. `start` remakes the copy, builds every program in it and runs `loader`; `reset` deletes it.
- `.sandbox` starts with a dot, so the toolchain run from the repository root ignores the copies' packages.
- A lock on `.sandbox/<instance>.lock` is held by the run and by the emulator it starts, which outlives `nan` by a moment after a termination request. A start or reset of a running instance waits up to 5 seconds for it, then fails.

## Consequences

- Each start copies the working tree; the shared, content-addressed build cache keeps the copy's build incremental.
- Instances share nothing else: the emulator opens no port.
- An instance's images and boot volume outlive its run, until the next start or reset.
