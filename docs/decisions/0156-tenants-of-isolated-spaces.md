# 0156. Tenants of isolated spaces

Status: accepted

## Context

Chapter 09 places an instance in the isolated space of its principal and provenance class, and chapter 04 requires an indirect branch prediction barrier before a CPU runs a space of another principal than the last, and the CPU buffers cleared before every return to user mode. NaN runs a space through `nanos::isolate(cpu, tenant, services, ...)`: the CPU issues the barrier when it switches to a thread of another tenant, and clears its buffers before each entry to user mode. Every job and instance a space holds reaches the log, standard streams, random source and file system of the space's `services`, and a job there holds no capability (nan-language/nan#297), while each NaN OS instance logs as itself, draws randomness only under a grant, may run on a taped clock and random source, and each job writes its own streams.

## Decision

- **One space per instance or job.** The core isolates each instance and job in a space of its own, with the services of its world, until NaN gives each job and instance of a space services of its own. Events and `/live` name the space by the label of its principal and class all the same.
- **Tenants.** The core numbers each principal whose code runs isolated, from 2 in the order its first space starts, and runs each of its spaces as that tenant; the `nan` job's toolchain runs as tenant 1.

## Consequences

- The spaces of one principal share branch prediction state; those of different principals do not.
- A principal keeps its tenant until the boot ends.
- Each isolated instance or job pays a space of its own (N63).
