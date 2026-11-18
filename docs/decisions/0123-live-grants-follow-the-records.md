# 0123. Live grants follow the records

Status: accepted

## Context

The runner builds every Env from its live grants. They came from the kernel image's genesis and from the grants it creates with the instances `Control` creates, such as the `Clock` and `Device` grants of drivers, so a grant a plan wrote to `/sys/grants` never became live, a job or specification naming one could not start, and a removed grant stayed live. Chapter 09 has the core revoke a commit's removed grants before the commit is acknowledged. Drivers hold the root entries of what they minted, and a driver may itself wait on the store. Plans number new grants after every grant `/sys/grants` holds, which knows nothing of the grants the runner creates.

## Decision

- **The store reports grants.** At a mount, before it tells its component it is mounted, the store reports every grant of `/sys/grants` on `main`; after each commit to `main` writing `/sys/grants/g<n>`, before answering it, it reports each such grant as `main` holds it, or its removal. It waits until the core holds them.
- **The core follows.** The runner adds or replaces the grants reported and removes the removed ones with their descendants, and at a mount every live grant below the runner's own range that the report lacks; it answers, then asks every driver to revoke what it minted of the removed grants without waiting for their answers.
- **The runner's own range.** Grants the runner creates take numbers from 2^48, which `/sys/grants` never reaches.

## Consequences

- A grant is live once the apply writing it answers, and after a boot the live grants are those the records describe besides the runner's own.
- A capability minted from a removed grant stays usable until its driver takes the revocation.
- Drivers' grants are not records of `/sys/grants`.
