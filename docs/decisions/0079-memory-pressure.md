# 0079. Memory pressure

Status: accepted

## Context

Chapter 05 gives each pool a pressure level from 0 to 4 against watermarks of its free memory, changed only when a draw or a return carries the free memory across a threshold, and arming no timer. Each rise calls `pressure` on the pool's instances; at level 3 the pool's idle checkpointed instances hibernate, least recently messaged first, and hibernation memory drops its images, oldest first, keeping none while the level stays at 3 or more; at level 4 starts fail with `budget.memory-pressure`. NaN tells no kernel when a draw or a return moves a cell's charge (nan-language/nan#197), and counts a component's delayed messages with its keyed tasks when it tells whether it is idle (nan-language/nan#200). The memory keeper settles the pool's level as it reads the pool's charge. The holders take their orders through bounded lists, whose `pop` takes the last element.

## Decision

- **Observed levels.** The memory keeper settles the pool's level at each request it takes: each start, which asks for the level, each domain's end, each hibernation and each read of `/live/memory`. A runner task waits for the level to change.
- **Rises.** At each rise, the holder of every running instance not paused calls `pressure` with each level risen through, in order, between handler calls.
- **Reclaim.** At level 3 or more, the runner drops the images hibernation memory keeps, oldest first, then hibernates the idle checkpointed instances, least recently messaged first, until the level falls below 3. Such a hibernation ends with the instance running when the component is not idle once it has handled what waits, and pressure then passes the instance over until a call reaches it. A hibernation at level 3 or more keeps no image.
- **The top level.** At level 4, a start whose cell the pool would supply fails with `budget.memory-pressure`, its instance `failed` with that fault.
- **Queues in order.** A holder takes its orders, and a domain hands its replies, in the order they were queued.

Alternatives rejected:

- **A timer reading the pool's charge:** it would wake an idle machine.
- **Reading the charge after every handler call:** one more request to the keeper for each handler call.

## Consequences

- Draws between two observations raise no level: an instance growing its state within its handler calls is seen at the next start, end or read.
- An instance with a pending `after` message is never idle for pressure.
- `store` does not implement `Pressure`, so level 1 drops none of its caches; no extra pool is made at level 4.
- The level followed is that of the first node pool.
