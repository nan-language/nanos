# 0098. Memory pressure

Status: accepted

## Context

Chapter 05 gives each pool a pressure level from 0 to 4 against watermarks of its free memory, changed only when a draw or a return carries the free memory across a threshold, and arming no timer. Each rise calls `pressure` on the pool's instances, and level 1 has `store` drop the clean entries of its caches, keeping the blob pieces a view or a mapping holds; at level 3 the pool's idle checkpointed instances hibernate, least recently messaged first, and hibernation memory drops its images, oldest first, keeping none while the level stays at 3 or more; at level 4 starts fail with `budget.memory-pressure`. An instance is idle when its intake is empty, no handler runs and it has no keyed task. NaN wakes a task once draws and returns take a cell's charge out of the band its watch's thresholds give, counts a stepper's keyed tasks apart from its delayed messages, and tells whether a shared value has another handle. The memory keeper settles the pools' levels as it reads their charges. The holders take their orders through bounded lists, whose `pop` takes the last element.

## Decision

- **Observed levels.** Each pool's root cell has a watch whose thresholds are the charges at which its free memory falls below a level's entry or reaches that entry and half a watermark. A core task waits on each watch and tells the memory keeper of each crossing; the keeper settles every pool's level then and at each request it takes. A runner task waits for the levels to change.
- **Rises.** At each rise, the holder of every running instance not paused calls `pressure` with each level risen through, in order, between handler calls. `store` then has its sequencer drop the blob pieces whose shared bytes no mapping holds after the batch it serves.
- **Reclaim.** At level 3 or more, the runner drops the images hibernation memory keeps, oldest first, then hibernates the idle checkpointed instances, least recently messaged first, until the level falls below 3. Such a hibernation, which keeps the messages the instance delays in its image, ends with the instance running when keyed tasks of it are held or running once it has handled what waits, and pressure then passes the instance over until a call reaches it. A hibernation at level 3 or more keeps no image.
- **The top level.** At level 4, a start whose cell the pool would supply fails with `budget.memory-pressure`, its instance `failed` with that fault.
- **Queues in order.** A holder takes its orders, and a domain hands its replies, in the order they were queued.

Alternatives rejected:

- **A timer reading the pools' charges:** it would wake an idle machine.
- **Reading the charges after every handler call:** one more request to the keeper for each handler call.

## Consequences

- Each pool costs a core task waiting on its watch.
- `store` keeps no cache of index nodes apart from its index, so level 1 drops blob pieces alone.
