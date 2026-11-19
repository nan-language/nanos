# 0128. Job entries, pacing and interruption

Status: accepted

## Context

Chapter 06 runs a host-compatible entry with the job's arguments and environment and keeps its exit status, runs a bundle of provenance `agent` on a serial executor and a 1 ms clock unless grants confer `exec.parallel` or `clock.fine`, and records `Interrupted` at the next boot for a job a stop ended. NaN runs a host-compatible entry through `nanos::run` with the encoding of a `nanos::HostInput` as its input and gives its exit status as the encoding of a `u8`. Only a bundle's admission record tells its entry and provenance. Jobs have no specification in `/sys/instances`, and the runner keeps no record of a boot's jobs.

## Decision

- **Entry from admission.** Admission records whether `main` takes a `host::World`. Each run's preparation reads the bundle's admission record for that and for its provenance.
- **Host-compatible entries.** The job order holds the encoding of the `HostInput` of the specification's arguments and environment, which such an entry takes as its input. Its result decodes as the exit status, and its branch merges only on 0.
- **Pacing.** A start of a job of provenance `agent` draws a seed and runs the job on `Executor.serial`, unless an `Exec` grant confers `parallel`, and coarsens its clock to 1 ms, unless a `Clock` grant confers `fine`. The run's result keeps the seed.
- **Interrupted.** At its start, the runner lists `/state` through a View for `system`, answered once `store` serves, and gives `Interrupted` to each job with a specification and no result that no driver of this boot runs, started when its specification was written.

## Consequences

- An interrupted job keeps its subtree of `/state` until it expires.
- A job whose specification was written in a boot that never ran it is also marked `Interrupted`.
