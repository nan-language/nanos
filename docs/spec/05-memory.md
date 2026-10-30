# Memory

This chapter defines the frame allocator, pools, mappings, cells and limits, reserved instances, the core reserve, pressure, stacks, mapped blobs, the charging of transfers, hibernation memory and the memory summary.

## Frames

**RAM** is the machine's usable memory as [NaN 24 §Anchor](https://github.com/nan-language/nan/blob/main/docs/spec/24-kernel.md#anchor) defines it. The **frame allocator** holds every region that `k.regions()` returns as frames of 4 KiB, 2 MiB and 1 GiB ([NaN 24 §Physical memory](https://github.com/nan-language/nan/blob/main/docs/spec/24-kernel.md#physical-memory)), per NUMA node; a machine whose firmware reports no nodes has one node.

- Frame metadata takes 16 bytes per 64 KiB of memory and is built for each GiB when that GiB is first used.
- The frame allocator gives frames to page tables, IOMMU tables, [pools](#pools) and [reserved instances](#reserved-instances). Every frame is zeroed between two owners. Each CPU keeps up to 16 zeroed 4 KiB frames for page tables, which it refills as it goes idle.

An idle frame allocator costs nothing.

## Pools

A **pool** is a NaN memory store over one region that the frame allocator gives up ([NaN 12 §Regions](https://github.com/nan-language/nan/blob/main/docs/spec/12-memory.md#regions)): the root cell of the store supplies, in 64 KiB chunks ([NaN 12 §Budgets](https://github.com/nan-language/nan/blob/main/docs/spec/12-memory.md#budgets)), every cell split below it, and the region stays the pool's for the rest of the boot. Every NaN allocation outside the boot heap and the regions of [reserved instances](#reserved-instances) draws from a pool, code areas included ([loading](10-code.md#loading)).

| Pool | Region | Supplies |
| --- | --- | --- |
| the core pool | part of the [core reserve](#core-reserve) | the core's cells: journal rings, runner records, capability tables |
| a node pool | at start, the largest free range of each NUMA node | instances, principals' sessions and hibernation images |
| an extra pool | a free range of at least 64 MiB, made when a node pool reaches [pressure](#pressure) level 4 | instances and sessions that start after it |

- The node pool of the node with the most memory leaves RAM ÷ 16 of its range to the frame allocator, which keeps it with every other free range for reserved instances, page tables beyond the core reserve and extra pools.
- At each start, revival and update of a top-level instance, and when a principal's first session opens, the core splits its cell from the pool with the most free memory among the pools of the node of the CPU that runs the start, or of any node when that node has none.
- An idle pool costs nothing.

## Mappings

- An instance in the [shared space](04-kernel.md#shared-space) reaches its chunks at their physical addresses: a draw or a return changes no mapping.
- An instance in an [isolated space](04-kernel.md#isolated-spaces) draws chunks with the `draw` [system call](04-kernel.md#system-calls). The core zeroes each chunk, then maps it into the space's user window read-write and non-executable, 32 contiguous chunks aligned to 2 MiB as one 2 MiB page. `release` unmaps the chunks, and they return to their pool once the [shootdown](04-kernel.md#switches-and-shootdowns) completes.
- A store blob is mapped read-only and shared by its readers ([mapped blobs](#mapped-blobs)).
- DMA buffers are NaN boxes in chunks ([NaN 23 §DMA](https://github.com/nan-language/nan/blob/main/docs/spec/23-low-level.md#dma)); on a machine with `iommu` they are mapped into the device's IOMMU domain ([DMA and IOMMU](15-devices.md#dma-and-iommu)).
- Nothing is paged. Every mapping exists before its first use, no memory is committed on a fault, and memory leaves RAM for storage only through [hibernation](#hibernation-memory).

## Cells and limits

Memory quotas are NaN cells only ([NaN 12 §Budgets](https://github.com/nan-language/nan/blob/main/docs/spec/12-memory.md#budgets)). The cells of a node pool form this tree:

```text
node pool          root cell of the pool's store
├── hibernation    hibernation images, at most a sixteenth of the pool
├── app            a top-level instance: InstanceSpec.budget.limit, with its intake cell
│   └── app.mail   a nested instance, split from its supervisor's cell
├── store          its own cell, with its store cache cell and blob cache cell
└── agent.ops      a principal's sessions: its memory budget
    └── s12        one session
```

- An instance's cell has the limit of its specification's `budget` ([`/sys`](13-tree.md#sys)). A top-level instance's cell is split from a pool's root cell ([Pools](#pools)), a nested instance's from its supervisor's cell.
- The runner draws the intake entries and encodings of an instance's waiting messages ([credits](07-communication.md#credits)) through an intake cell split from the sender's cell, so the sender's limit bounds them.
- A draw is charged to every cell of its chain, and a draw exceeding any limit of the chain is exhaustion where it is made ([NaN 12 §Budgets](https://github.com/nan-language/nan/blob/main/docs/spec/12-memory.md#budgets)).
- The limits of the cells of one pool may sum to more than the pool's memory.
- A changed limit takes effect at the instance's next start, revival or update, when the core splits its new cell.
- An active instance holds at least one chunk; a hibernated one holds at most 2 KiB ([hibernation memory](#hibernation-memory)).
- A principal's sessions draw through one **principal cell**, split when its first session opens, whose limit is `SessionBudget.memory_bytes` of its [session budget](18-link.md#budgets), or the pool's size when that is `None`. Each session's cell is split from it, and previews and other work charged to the session draw through the session's cell.
- The memory of an [account](08-scheduling.md#accounting) is the charge of its instance's or session's cell.

The `budget` field of an instance specification:

```nan
pub closed struct Budget derives Copy, Eq, Encode {
    pub limit: u64 where limit >= 65_536 and limit % 65_536 == 0
    pub reserved: bool
} Budget
```

`limit` is in bytes, by default 16 MiB (`Budget.limit`). `reserved` is false by default (`Budget.reserved`).

## Reserved instances

An instance whose specification sets `Budget.reserved` has memory that no other draw takes:

1. At its first start, the core takes ⌈limit ÷ 2 MiB⌉ contiguous 2 MiB frames from the frame allocator as one region, makes a NaN memory store over it and gives the instance the store's root cell as its allocator. Failure: when no free range is large enough, the start fails with [`budget.over-reservation`](18-link.md#codes).
2. The instance's draws come from its region only. Pressure never fails them and never hibernates the instance.
3. The region stays set aside for the rest of the boot. A restart reuses it; once the instance is removed, the core gives the region to the next reservation that fits in it.

The limits of reserved instances sum to at most RAM minus the core reserve: a plan whose effect exceeds that sum is refused ([`budget.over-reservation`](18-link.md#codes)).

## Core reserve

The **core reserve** is 8 MiB + 256 KiB per CPU + the frame metadata. The core sets it aside at [start](04-kernel.md#start-sequence) step 3: the frame metadata, frames for page tables and IOMMU tables, and the core pool, of 4 MiB + 256 KiB per CPU. Only the core draws from it. Page tables beyond its frames come from the frame allocator's free memory; when it has none, the instance whose start needs them fails to start as [starting](06-execution.md#starting) states for a refused draw.

On the sandbox, with 4 CPUs and 512 MiB, the core reserve is 8 MiB + 1 MiB + 128 KiB.

## Pressure

The **watermark** of a pool is w = 4·√(its size in KiB) KiB, rounded down to a whole KiB. A pool's **free memory** is the memory its root cell has not drawn. Each node pool and extra pool has a **pressure level** from 0 to 4, with entry thresholds T(1) = 4w, T(2) = 3w, T(3) = 2w and T(4) = w:

| State | Event | Next | Effect |
| --- | --- | --- | --- |
| level n, n < 4 | the pool's free memory falls below T(n + 1) | level n + 1 | the action of level n + 1 starts |
| level n, n > 0 | the pool's free memory reaches T(n) + w/2 | level n − 1 | the action of level n stops |

| Level | Action, added to those of the levels below |
| --- | --- |
| 1 | the runner calls `pressure(1)` on `store` when the pool holds its cell; `store` drops the clean entries of its store cache and blob cache |
| 2 | it calls `pressure(2)` on every instance of the pool whose state implements `Pressure` |
| 3 | it calls `pressure(3)` on them, and the pool's idle checkpointed instances hibernate in the order of [`mem.reclaim`](04-kernel.md#kernel-policies), least recently messaged first by default |
| 4 | it calls `pressure(4)` on them, and the core makes an extra pool when the frame allocator holds a free range of at least 64 MiB beyond the core reserve; while every pool is at level 4, instance starts and limit increases fail with [`budget.memory-pressure`](18-link.md#codes) |

- w is 1 MiB for a pool of 64 MiB, 2,896 KiB for 512 MiB, 16 MiB for 16 GiB and 128 MiB for 1 TiB.
- The core changes a pool's level only when a draw or a return carries its free memory across a threshold; pressure arms no timer.
- Hibernation for pressure goes on while the pool's level is 3 or more and an idle checkpointed instance of the pool remains that it does not pass over. It passes over an instance whose [hibernation](06-execution.md#hibernation) ended with the instance `running`, until one of its handlers runs or its next checkpoint is durable.
- A draw that fails, at any level, fails where it is made: the drawing instance traps (`trap.exhausted`) or handles `mem::OutOfMemory` ([NaN 12 §Growth](https://github.com/nan-language/nan/blob/main/docs/spec/12-memory.md#growth)). No instance is stopped to give its memory to another.
- `Pressure` is a [system interface](06-execution.md#system-interfaces) (requires [N1](22-nan.md#n1)): the runner calls `pressure(level)` as a handler call at each rise of the pool's level, with that level.

## Stacks

| Code | Stack | Charged to |
| --- | --- | --- |
| tasks with a static bound | exactly that bound ([NaN 17 §Static bounds](https://github.com/nan-language/nan/blob/main/docs/spec/17-suspension-io.md#static-bounds)), without a guard region | the cell of the task's domain (requires [N8](22-nan.md#n8)) |
| interrupt fns | the CPU's interrupt stacks | the kernel image |
| space threads in the core | the kernel stack that NaN 24 sizes for threads running user code | the core reserve |
| tasks in an isolated space | user stacks that the user-mode runtime draws with `draw` (requires [N23](22-nan.md#n23)) | the cell of the task's domain |
| tasks without a static bound, in bundles for the nanos targets | 64 KiB at start, grown by the runtime (requires [N1](22-nan.md#n1)) | the cell of the task's domain |

A stack never overflows: a static bound fixes its size, and a grown stack is drawn like any memory, so its domain's limit bounds it.

## Mapped blobs

A **mapped blob** is a store [blob](11-store.md#nodes) read in place, without a copy:

- Its 64 KiB pieces lie in the blob cache ([blob pieces](12-store-format.md#blob-pieces)), one chunk each, once whatever the number of readers.
- A reader in the shared space reaches the pieces through a read-only byte view ([transfer](07-communication.md#transfer)) (requires [N27](22-nan.md#n27)). A reader in an isolated space gets them mapped read-only and non-executable into its window.
- A piece stays in the cache while a view or a mapping holds it; pressure level 1 drops only pieces nothing holds.
- The pieces are charged to the blob cache's cell, never to readers. A mapping costs its reader only page tables.

## Transfers

[Transfer](07-communication.md#transfer) and the [isolated transport](07-communication.md#isolated-transport) carry messages; their memory is charged as follows:

- Within a bundle, a message's value moves ([NaN 11 §Domain memory](https://github.com/nan-language/nan/blob/main/docs/spec/11-errors-domains.md#domain-memory)). Each chunk the value alone occupies, as every allocation larger than 64 KiB does, passes with its charge to the receiver's domain and cell when the message is delivered (requires [N7](22-nan.md#n7)). A chunk the value shares with other allocations passes to the innermost domain holding both instances and stays charged to the sender's cell while any of its allocations lives.
- Between bundles or spaces, a waiting message's intake entry and encoding are drawn through the sender's intake cell and released when a handler call takes the message, whose decoded value is drawn through the receiver's cell. The frames of a core transfer are charged to the sender's cell and pass to the receiver's when a handler call takes the message.
- A channel's frames are charged to its sender's cell for as long as the channel exists.

## Hibernation memory

[Hibernation](06-execution.md#hibernation) leaves an instance its intake, its runner tables and its pending `after` messages, at most 2 KiB besides its waiting messages and its published summary; the runner builds its Env again at revival ([Env construction](09-authority.md#env-construction)). It keeps the hibernation image as follows:

- It compresses the image in the LZ4 block format, and keeps it uncompressed when compression does not shrink it.
- It keeps the image in the hibernation cell of the instance's pool while that cell stays within a sixteenth of the pool and the pool's level is below 3. Otherwise it keeps no copy in RAM: hibernation has checkpointed the state, so the latest [checkpoint](06-execution.md#checkpoints) stands as the image.
- When the pool's level reaches 3, it drops the pool's images, oldest first, for as long as the level stays at 3 or more; each dropped image's checkpoint stands as the image.
- Revival decodes the image, or the checkpoint standing as it, into chunks drawn through the instance's new cell, and keeps the incarnation; a refused draw fails the revival as [starting](06-execution.md#starting) states.
- A `/live` read of a hibernated instance returns its last published summary and never revives it ([`/live`](13-tree.md#live)).

## Memory summary

`/live/memory` holds one record whose byte counts partition RAM: the fields other than `ram` and `level` sum exactly to `ram`. It is computed when read from counters that the frame allocator and the cells keep, so reading it wakes nothing.

```nan
/// The memory summary at /live/memory, in bytes.
pub closed struct MemorySummary derives Encode {
    /// Usable memory.
    pub ram: u64
    /// The kernel image and the memory it reserves, the boot heap excepted.
    pub image: u64
    /// NaN's boot heap, drawn or not.
    pub boot_heap: u64
    /// Usable memory that no region or frame covers.
    pub excluded: u64
    /// The frame allocator's metadata.
    pub frame_metadata: u64
    /// Frames of page tables and IOMMU tables.
    pub page_tables: u64
    /// The core pool, drawn or not.
    pub core: u64
    /// Chunks of the intake cells.
    pub intakes: u64
    /// Hibernation images kept in RAM.
    pub hibernation: u64
    /// The store's cache of index nodes and records.
    pub store_cache: u64
    /// The store's cache of blob pieces.
    pub blob_cache: u64
    /// Reserved regions not drawn.
    pub reserved: u64
    /// Chunks drawn through instance, principal and session cells, the caches and intake cells excepted.
    pub accounts: u64
    /// Free frames, and the memory of node and extra pools their root cells have not drawn.
    pub free: u64
    /// Memory between owners: domains in settlement, chunks on their way back to their pools.
    pub unattributed: u64
    /// The highest pressure level of the pools.
    pub level: u8 where level <= 4
} MemorySummary
```

Each instance's and session's node under `/live/accounts` gives its cell's charge, peak charge and limit ([`/live`](13-tree.md#live)).
