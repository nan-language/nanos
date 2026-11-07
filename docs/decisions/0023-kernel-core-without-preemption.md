# 0023. Kernel core without preemption

Status: accepted

## Context

NaN's kernel runtime owns what chapters 04 and 08 assign to the core at the lowest level: it starts the CPUs, keeps one run queue per CPU ordered by a key, switches tasks, keeps the clock and the timers, and idles CPUs. The core supplies an `hw::Policy` whose hooks place each task that becomes ready on a CPU with a key, bound to that CPU or not, name the CPU an idle one steals from and how many of its unbound tasks, give slices and timer slack, and choose how a CPU idles; they run with interrupts disabled, `notrap` and `noalloc`, and see a task only as an `hw::TaskView`. NaN enables no-execute pages and kernel write protection on every CPU, and clocks the kernel by the time-stamp counter where it is invariant, else by the HPET or the ACPI PM timer.

`hw::Ring0` is linear: one value performs every privileged operation on the CPU running its holder. The CPUs the anchor starts run on NaN's entry root until the core installs its own.

`boot::Kernel` hands out usable memory as `hw::Region`s, which `hw::Region.split` divides, and gives the memory map and the extents of the image's segments, the boot area and the start information. Its PCI functions and ports, their register windows and interrupts, and the kernel clock reach device registers at their physical addresses, which NaN's entry root maps uncached across the first 4 GiB, and its sandbox machine places every BAR there.

NaN v0.6.0 loses a wake-up across a preemption: a task preempted after it registers to wait and before it parks resumes as running, the wake that came meanwhile forgotten, and then parks for good. A wake preempts the running task when the woken one is placed with a smaller key, and a slice preempts it when it ends. NaN's default policy, under which its own tests run, keys tasks by the order they become ready and gives no slice, so it never preempts. In the sandbox, instance tasks waiting for the core's answers stalled at random places of the boot.

## Decision

- **The first task holds `hw::Ring0`.** Work that must run on a given CPU runs as a core task of an executor pinned to that CPU: its share's class byte marks it pinned and its `reserve_ppm` names the CPU, where the policy places it bound. Such tasks borrow `hw::Ring0` from the first task's scope. Starting a CPU is one: it enables the CPU's features, installs its root and marks it started; until then the policy places no other work on it, and no CPU steals before every CPU has started.
- **Tasks run in the order they become ready, never preempted.** Each task is keyed by the time it becomes ready and gets no slice, so a wake never preempts. Pinned and `latency` work is placed bound to its CPU; an idle CPU steals half the unbound tasks of the busiest queue sharing its last-level cache, else its node, else the machine. Wake placement and stealing read per-CPU state that the hooks and the core's tasks share behind spin locks. Idle CPUs halt. A timer's slack follows the class of the work arming it as chapter 08 gives it, and a fifth class, deferrable core work, gets a window that never ends first: its timers pass with the next firing and never wake an idle machine.
- **Time is NaN's.** The monotonic clock is the kernel clock, and timers are the runtime's deadlines.
- **Regions are kept whole.** The largest region of each node becomes its pool, the smallest other region holding the core pool becomes it, and the rest go to the frame allocator, as 4 KiB frames merged into 2 MiB and 1 GiB frames where aligned; its first 1,024 4 KiB frames are the core reserve's for page tables. The pool of the node with the most memory leaves the top RAM ÷ 16 of its region, from a 2 MiB boundary, to the frame allocator as a range it keeps whole, from which reserved instances take their regions.
- **The shared space maps what `k.regions()` bounds, and device memory as the entry root does.** Each CPU has a root of its own reaching the same lower tables. Below 4 GiB the shared space maps everything up to the end of the last region there, the gaps between regions holding the image, the boot area and the start information, and maps the image's span executable; it maps the rest of the first 4 GiB uncached, where the PCIe configuration windows, the BARs the firmware and NaN assign, the I/O APIC, the HPET and the local APIC lie, which NaN's runtime and drivers reach through the windows it checks against the entry root; above 4 GiB it maps the regions alone. It maps the first MiB, where the anchor starts the other CPUs, until every CPU has started, then removes it with a shootdown.
- **Firmware tables are copied before the shared space is installed**, which maps none of the firmware's own memory, and decoded by the `acpi` package.
- **The first task prints the start line** with the measured facts and the core's memory, the one observation of a started kernel until the link exists.

Alternatives rejected:

- **`hw::Ring0` in the anchor:** only interrupt and landing fns reach the anchor, so no task could shoot down translations.
- **Starting each CPU from an interrupt fn:** the CPU signal would be one more IPI than chapter 04 allows, and the fn would need `hw::Ring0`.
- **Keys ranking the classes, with a virtual runtime per account and slices:** wakes and slices would preempt, and lose wake-ups.
- **Guarding each wait with a timer:** every wait of every instance would need it, and a guarded wait still stalls until its timer.
- **Mapping the regions alone:** the boot area, which holds every task's stack, would be unmapped.
- **Splitting regions into 2 MiB frames:** their edges would be lost for the boot.
- **Mapping each device's windows as its driver claims them:** the runtime reads PCIe configuration windows and the I/O APIC under whatever root is installed, and maps no window itself.

## Consequences

- Chapter 08's classes order no queue and `normal` accounts do not share a CPU by weight: a task runs until it suspends, and a CPU-bound task holds its CPU. Ranked keys and slices come back once NaN keeps a wake-up across a preemption.
- The shared space maps holes below 4 GiB and maps the image writable and executable, device registers above 4 GiB are out of reach, the core pool is the whole region holding it, and reserved instances take their regions only from the range the richest pool leaves; CPUs with `mwait` idle in HLT. NaN offers the memory map, region splits and `mwait`, and adopting each changes the start's memory layout or the idle path: each is a change of its own.
- Splitting a region into 4 KiB frames draws a list of all of them through the boot heap, 2 MiB per GiB, while the frame allocator takes the region.
