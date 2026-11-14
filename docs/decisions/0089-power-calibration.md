# 0089. Power calibration

Status: accepted

## Context

Chapter 08 charges energy as CPU time × the power of a running CPU, calibrated from RAPL at most once per 10 ms per running CPU, and `PowerConfig.busy_mw` without it. Every RAPL CPU counts its package's energy; only some count its cores' apart. Reading an MSR a CPU lacks faults, and NaN offers no read that survives the fault. The policy's hooks run with interrupts disabled and cannot read MSRs; the runner charges every account at one machine-wide power. An idle machine arms no timer.

## Decision

- **Source.** The package energy counter, on AMD and Hygon CPUs whose leaf `0x8000_0007` reports RAPL and on the Intel family 6 models known to count it in units of 2^-n joules.
- **Pace.** The policy has a CPU switching to work ask for a calibration once 10 ms passed since it last asked; a core task pinned to that CPU reads its package's counter. No calibration runs while the CPU idles.
- **Estimate.** Each sample of a package ends a window since its previous one: the energy counted over the time the package's CPUs spent on work, taken when that work filled at least half of a window of at most 60 s. The first estimate replaces `busy_mw`, each later one moves it an eighth of the way.

## Consequences

- A package's whole energy, idle CPUs and uncore included, is charged to the work it ran.
- A lightly loaded machine keeps the estimate of its last busy window.
- A CPU that counts RAPL but is missing from the model list charges at `busy_mw`.
