# 0159. Isolated spaces run under split roots on every machine

Status: accepted

## Context

NaN runs a space's thread under a root mapping only its user window and the entry area, tagged on each CPU with a tag of its own. NaN OS asked that on CPUs Meltdown does not affect a space's root share the kernel's tables below its window, saving the root switch of each crossing (N61). NaN keeps the root split on every CPU: the kernel code serving a space's system call runs after the thread's exit, so one root would keep the space's tables in the CPU's walks after the space is released and reach every space's tag on each invalidation of the kernel's translations (nan-language/nan#291).

## Decision

A space's root maps only its user window and the entry trampoline on every machine, and the core runs under the shared space's root. Meltdown immunity is no machine condition.

## Consequences

- Each crossing between an isolated space and the core switches roots twice, flushing nothing on a machine with `pcid`.
