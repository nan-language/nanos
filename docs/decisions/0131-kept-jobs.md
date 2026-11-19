# 0131. Kept jobs

Status: accepted

## Context

Chapter 06 keeps a job's `/state/<moniker>/` subtree, and its branch when nothing merged it, `keep_h` hours after its end, and frees its moniker then; chapter 11 expires a kept job branch after those hours. The store gives a branch its expiry only when it creates it, before the job's end is known. The runner forgets a job's moniker when the job ends, and numbers unnamed jobs from 1 at each boot. A job a stop of the machine ended has no driver at the next boot.

## Decision

- **`keep_branch`.** The Store protocol gains `keep_branch(name, until)`, which sets a branch's expiry and needs `admin`. At a job's end the runner keeps its branch until `keep_h` hours later when nothing merged it.
- **Taken monikers.** A moniker is in use while `/state/<moniker>` exists: the preview of a `run` naming it fails with `instance.moniker-taken`.
- **Boot sweep.** The sweep giving interrupted jobs their result also has their subtree expire and their branch kept `keep_h` hours from the boot, and continues the numbering of unnamed jobs past the highest `job-<n>` that `/state` holds.

## Consequences

- A job that takes a moniker between another `run`'s preview and its apply makes the runner refuse that `run` only while it runs.
- Expiry counts from the boot that found the job interrupted, not from the stop that ended it.
