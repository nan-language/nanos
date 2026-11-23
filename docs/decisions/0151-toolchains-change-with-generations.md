# 0151. Toolchains change with generations

Status: accepted

## Context

NaN builds a bundle only against an interface file recording the building toolchain's hash, and keeps that rule: lowered code inlines the runtime's record layouts and lays out what crosses a gate as the kernel's toolchain does, an ABI NaN leaves unspecified ([nan-language/nan#283](https://github.com/nan-language/nan/issues/283)). Only the toolchain that built a kernel lowers bundles for it. A host build of the toolchain peaks at about 1.7 GiB, more than the 768 MiB heap of the `nan` job's space, and its bundle file, over 80 MB, exceeds the 64 MiB a `put` carries.

## Decision

- A machine's `nan` job runs its generation's toolchain, for checking and lowering alike.
- A newer toolchain comes with a new generation, whose kernel it builds; the generation switch lowers with it the images the generation lacks.
- `admission` has no operation lowering admitted bundles again for the running kernel.

Alternatives rejected:

- **Lowering again after a toolchain update:** the running kernel loads only the images of the toolchain that built it.
- **An updated toolchain checking at once while the generation's lowers:** a machine can neither take the toolchain's bundle through the link nor check and lower it itself, so it would run a toolchain it never checked.

## Consequences

- Moving a machine to a newer NaN release takes a generation switch.
- A machine holds one toolchain, its generation's.
