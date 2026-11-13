# 0085. Energy-performance preferences

Status: accepted

## Context

Chapter 08 has a CPU with hardware-controlled performance states (HWP) set its energy-performance preference to the lowest among the classes it runs, writing it only when it changes and at most once per 10 ms. The preference is a field of a model-specific register, which only code holding `hw::Ring0` writes. NaN's policy hooks, which see each CPU's classes, hold no `hw::Ring0` and run with interrupts disabled, but a task a hook makes ready joins that CPU's queue.

## Decision

- **Choice.** The scheduler counts the `latency`, `normal` and `background` tasks queued on each CPU. As the CPU switches to a task, it takes the lowest preference among those classes and the one it switches to, and posts it to the CPU's signal when it differs from the one it last posted. Core work counts for no class.
- **Write.** On a machine whose CPUs report HWP with its preference field, the core runs a task pinned to each CPU that hands the CPU's performance states to its hardware, then writes each preference posted that differs from the one written, over the CPU's whole performance range, and waits 10 ms after each write, so that the latest preference posted meanwhile is the next one written.
- Without HWP, no signal or task exists and the firmware's setting stays.

## Consequences

- A preference reaches the hardware one task switch after the policy chose it, and no timer runs while it does not change.
- A CPU whose queued tasks another CPU steals counts them until it next goes idle.
