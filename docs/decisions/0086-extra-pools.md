# 0086. Extra pools

Status: accepted

## Context

Chapter 05 has the core make an extra pool from a free range of at least 64 MiB when a pool reaches pressure level 4, split each new cell from the pool with the most free memory, and refuse starts only while every pool is at level 4. Each pool has its own level and its own actions: `pressure` calls on the instances drawing from it, and at level 3 the hibernation of its idle checkpointed instances. Hibernation images are kept in the node pool's hibernation cell. The memory keeper holds the frame allocator and the pools' cells; the runner reaches it through `runner::Machine`.

## Decision

- **Pools.** The memory keeper holds the node pool and the extra pools, numbered from 0 for the node pool. At each request it settles every pool's level, and when one has just reached level 4 it takes a 64 MiB region from the frame allocator's ranges for reserved regions and makes it an extra pool, for the rest of the boot. The region does not count as a reserved region in the memory summary.
- **Cells.** `Machine.pool` names the pool a new cell splits from: of the pools below level 4, the one with the most free memory, the lowest numbered on ties. The runner keeps that number with the instance's driver. A fork's cell comes from the pool named when the fork is made.
- **Pressure.** `Machine.changed` gives every pool's level, and the runner acts on each pool's rises: `pressure` calls on the running instances drawing from that pool, and from level 3 the hibernation of its idle checkpointed instances, after the hibernation images when the pool is the node pool. Reserved instances draw from no pool.

## Consequences

- An extra pool holds no hibernation images; the instances of an extra pool keep theirs in the node pool's hibernation cell.
- A pool whose level reaches 4 again makes another extra pool, as long as the frame allocator holds a free 64 MiB.
