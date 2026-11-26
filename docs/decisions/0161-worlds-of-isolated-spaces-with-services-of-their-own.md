# 0161. Worlds of isolated spaces with services of their own

Status: accepted

## Context

NaN places each job and instance of an isolation with services and capabilities of its own, its clock among them, and keeps the space's time by the clock of the services `isolate` takes: a world whose clock is revoked reads no time while its space and the space's other worlds keep theirs. An isolation draws the memory of every job and instance it places from the frames `isolate` takes, once, and a fault no frame serves ends each of them (nan-language/nan#311).

## Decision

- **A world's own services.** The core places each isolated job and instance with the clock, log, standard streams, random source and file system of its world, a job with its world's capabilities too, and makes its space with the same services but the core's monotonic clock, by which the space keeps time.
- **One space per job or instance.** The core makes a space for each job and instance, lending it the frames of that one alone, until an isolation takes the frames of each job and instance it places (N67).

## Consequences

- An isolated job or instance without a `Clock` grant reads no time (N65).
- A job in an isolated space holds the capabilities its grants give it; a call through one that its kernel takes time to answer ends it trapped as abandoned while nan-language/nan#314 stands.
- Supersedes 0158, and 0156's decision giving each instance and job a space of its own until NaN gives each job and instance of a space services of its own.
