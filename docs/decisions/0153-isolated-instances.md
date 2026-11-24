# 0153. Isolated instances and jobs

Status: accepted

## Context

Chapter 09 places an instance or a job at each start: in the shared space when admission placed its bundle there, it acts for `system` or `owner` and its spec does not ask for isolation, else in the isolated space of its principal and provenance class. NaN runs a job of a bundle in an isolated space through `nanos::run_isolated`, and holds an instance of a bundle's component in one through `nanos::hold_isolated`. Both take the machine's `hw::Ring0`, the bundle isolated through `Loader.isolate_unverified`, a link of user frames and the 4 KiB frames the space is made of; each space holds one instance or job (nan-language/nan#285), and its world holds only a clock, a log, standard streams, a random source and a file system (nan-language/nan#284). The core alone holds the `Ring0`, which is linear and only lent. A channel carrying frames has a linear receiver, which only an awaited drain consumes; its sender is droppable. The components of the kernel image run only in the shared space.

## Decision

- **Placement.** `os::Space.of` computes the space from the spec's principal and placement and its bundle's admission record, `shared` and provenance; the runner computes it at each start, and the link when it previews a `run` step. A component of the kernel image runs in the shared space, `keys` included until NaN carries the kinds of its Env into a space. `/live/instances/<moniker>` names the space `shared` or `<principal>/<class>`. On a machine without `user-mode`, the link refuses a `run` step placing its job isolated, and a start of an instance placed isolated fails with `place.unavailable`.
- **The core holds the space.** The task of the domain sends the bundle's image and its world's clock, log, standard streams, random source and file system to a core task through the `Spaces` capability, with the instance's checkpoint and runner capability, or the job's input. That task isolates the bundle and makes the space on the domain's executor and cell. It holds an instance through `hold_isolated`, handing the `nanos::Instance` to the domain's task, which drives it as it drives an instance of the shared space, until it releases it. It runs a job through `run_isolated` until the job ends or the domain's task, abandoned or cancelled, drops the sender of its stop.
- **Frames.** A space takes its memory limit, two copies of its bundle's image and 16 MiB for the runtime, its stack, its shared frames and its tables. The core task keeps a pool of frames: it borrows what a space lacks from the frame allocator, counted as reserved, and takes back the frames of each ended space for the next one. A start whose space finds too few frames ends with `budget.memory-pressure`.

Alternatives rejected:

- **The domain's task holding the space:** it has no `Ring0`, which only the core holds.
- **Giving each space's frames back to the frame allocator:** the memory keeper's requests would carry frames, so every channel of them would need a drain with a frame allocator at hand.

## Consequences

- An isolated instance or job whose Env holds a port, a View or another kernel kind fails at its start until N54.
- Each isolated instance or job pays a space of its own (N55).
- The frames isolated spaces once took stay reserved for later spaces until the boot ends.
