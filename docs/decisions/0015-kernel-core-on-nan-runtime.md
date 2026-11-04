# 0015. Kernel core on NaN's runtime

Status: accepted

## Context

NaN's kernel runtime owns what chapters 04 and 08 assign to the core at the lowest level: it starts the CPUs, keeps one run queue per CPU ordered by a key, switches tasks, keeps the clock and the timers, and idles CPUs. The core supplies an `hw::Policy` whose hooks place each task that becomes ready on a CPU with a key, name the CPU an idle one steals the first task from, give slices and choose how a CPU idles; they run with interrupts disabled, `notrap` and `noalloc`, and see a task only as an `hw::TaskView`.

`hw::Ring0` is linear: one value performs every privileged operation on the CPU running its holder. The CPUs the anchor starts run on NaN's entry root until the core installs its own.

`boot::Kernel` hands out usable memory as `hw::Region`s, which cannot be split, and gives no memory map and no extent of the image's segments, the boot area or the start information, all of which lie in usable memory that `k.regions()` leaves out. NaN enables `EFER.NXE` and `CR0.WP` only in kernels exporting a load ABI. `hw::Idle` offers halting and polling, not MWAIT. The kernel clock counts the TSC whether or not it is invariant, and timers fire at their deadlines, without slack. These gaps go to NaN; this ADR records what the core does meanwhile.

## Decision

- **The first task holds `hw::Ring0`.** Work that must run on a given CPU runs as a core task of an executor pinned to that CPU: its share's class byte marks it pinned and its `reserve_ppm` names the CPU, which the policy honours. Such tasks borrow `hw::Ring0` from the first task's scope. Starting a CPU is one: it enables the CPU's features, installs its root and marks it started; until then the policy places no other work on it, and no CPU steals before every CPU has started.
- **Keys carry the classes.** A key's top two bits rank `latency`, core, `normal` and `background` work; below them come the deadline, the arrival order of core work, or the account's virtual deadline. Each CPU keeps the virtual runtime of up to 64 accounts with ready work, folding each run into its account at the next switch, so `normal` and `background` accounts share a CPU by weight whatever number of tasks each has ready. Wake placement and stealing read per-CPU state that the hooks and the core's tasks share behind spin locks. Idle CPUs halt.
- **Time is NaN's.** The monotonic clock is the kernel clock, and timers are the runtime's deadlines.
- **Regions are kept whole.** The largest region of each node becomes its pool, the smallest other region holding the core pool becomes it, and the rest go to the frame allocator, as 4 KiB frames merged into 2 MiB and 1 GiB frames where aligned; its first 1,024 4 KiB frames are the core reserve's for page tables.
- **The shared space maps what the core cannot tell apart.** Each CPU has a root of its own reaching the same lower tables. Below 4 GiB the shared space maps everything up to the end of the last region there, the gaps between regions holding the image, the boot area and the start information, and maps the image's span executable; above 4 GiB it maps the regions alone. It maps the first MiB, where the anchor starts the other CPUs, until every CPU has started, then removes it with a shootdown. The core sets `EFER.NXE` on each CPU before installing its root.
- **Firmware tables are copied before the shared space is installed**, which maps none of the firmware's own memory, and decoded by the `acpi` package.
- **The first task prints the start line** with the measured facts and the core's memory, the one observation of a started kernel until the link exists.

Alternatives rejected:

- **`hw::Ring0` in the anchor:** only interrupt and landing fns reach the anchor, so no task could shoot down translations.
- **Starting each CPU from an interrupt fn:** the CPU signal would be one more IPI than chapter 04 allows, and the fn would need `hw::Ring0`.
- **A virtual runtime per task:** an account would get CPU time in proportion to its ready tasks rather than its weight.
- **Mapping the regions alone:** the boot area, which holds every task's stack, would be unmapped.
- **Splitting regions into 2 MiB frames:** their edges would be lost for the boot.

## Consequences

- Until NaN provides what it lacks: the shared space maps holes below 4 GiB and the image writable and executable, and kernel mappings are not write-protected; node pools take whole regions, so the node with the most memory leaves no share of its largest range to the frame allocator, and the core pool is the whole region holding it; CPUs with `mwait` idle in HLT; without `invariant-tsc` the monotonic clock counts the TSC; timers have no slack.
- A CPU stealing takes the first task of the queue it names, so the policy steals from no CPU while work pinned to it waits, which `latency` work bound to its reservation's CPU will need as well.
- Splitting a region into 4 KiB frames draws a list of all of them through the boot heap, 2 MiB per GiB, while the frame allocator takes the region.
