# 0129. Job streams and file views

Status: accepted

## Context

Chapter 06 gives a job's world its standard streams, reading the blob its specification names at `/state/<moniker>/stdin` and writing one entry per write of the streams `/state/<moniker>/stdout` and `/state/<moniker>/stderr`, each keeping 16 MiB, and a `sys::Fs` over the dir its grant `fs` names, on its branch; chapter 11 makes that `Fs` a file view whose files are blobs. The services behind a world's `Stdio` and `Fs` run on the job's tasks within 4 KiB of stack, and `release` of a file cannot wait. Store calls go through Views on tasks of their own.

## Decision

- **A serving task per run.** Each run of a job has a task serving its streams through the job's View of `/state`, and, when it holds an `fs` grant, one serving its file view through that grant's View. The services forward each call to their task over a channel and wait for its answer; `release` queues its order without waiting.
- **Streams.** The run's preparation declares `stdout` and `stderr` as streams of `mem::Bytes` keeping 16 MiB when the job lacks them, and places its input blob at `stdin`, in the commit writing its specification. Reads take the input blob in order; each write appends one entry.
- **Files.** A file opened to read reads its blob as of the commit its open saw; one created writes a draft, which its release puts at its path. `write_all`, `remove`, `create_dir`, `rename` and a release are each one blind commit, which the job's branch takes until the job's end merges it.
- **Revocation.** The run's watch revokes both services when the run ends, which ends their tasks; drafts left open are discarded.

## Consequences

- A release whose commit fails loses the file's bytes without telling the job.
- A job's streams stay in its subtree of `/state` until it expires.
