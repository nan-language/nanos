# 0078. Memory summary

Status: accepted

## Context

Chapter 05 publishes at `/live/memory` a `MemorySummary` whose byte counts partition RAM, computed when read from counters that the frame allocator and the cells keep, so reading it wakes nothing. A pool's free memory is what its root cell has not drawn, and its pressure level follows that free memory against a watermark of 4·√(its size in KiB) KiB. NaN gives a budget's charge and highest charge but nothing of a root cell: a store over a region tells neither what it has drawn nor what is left. Every draw from a node pool goes through a cell the core splits from it: an incarnation's cell, a fork's or the hibernation cell. NaN's `x86_64-bare` boot heap is 16 MiB.

## Decision

- **The pool's own cell.** The core splits from each node pool a cell of the pool's whole size and splits every other cell of the pool from it, so its charge is what the pool has drawn. The hibernation cell is one of them.
- **Fixed parts.** The start fixes RAM, the memory map's usable memory; the kernel image, the pages its segments span; the boot heap; the core pool, its chunks; and leaves as excluded what neither these, the node pools' chunks nor the frame allocator hold.
- **The memory keeper.** One core task holds the frame allocator and the pool's cells: it carves reserved instances' regions and answers readings of the frames page tables hold or the core reserve keeps for them, of the regions reserved instances hold, of the free frames, and of what the pool has drawn and its hibernation cell holds, settling the pool's level on its free memory as it reads.
- **The summary.** The board reads the charges of the running incarnations' cells, those of reserved regions apart, then the keeper's counters. What the pool has drawn beyond those cells and the hibernation cell is unattributed; counters read apart that overlap take the excess from free memory, so the counts always sum to RAM.

Alternatives rejected:

- **Summing the cells of instances as the pool's charge:** chunks that a closed cell still owns, those of values moved out of a freed domain, would count as free.
- **A periodic snapshot:** it would wake an idle machine.

## Consequences

- The frame allocator's records lie in the core pool, so `frame_metadata` is 0. Intakes and the store's caches draw through no cell of their own, so `intakes`, `store_cache` and `blob_cache` are 0 and their memory counts in `accounts`, as sessions' memory does in `link`'s.
- A fork's cell draws from the pool without a gauge, so its memory is unattributed until it ends.
- The level a reading settles is the one `/live/memory` shows; nothing else reads it yet.
