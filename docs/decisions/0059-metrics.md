# 0059. Metrics

Status: accepted

## Context

Chapter 17 has `journal` sample each second every account charged anything in it, keep 1-second samples 1 h, merge them into 60-second samples kept 24 h and those into 3,600-second samples kept `metrics_h`, adding counters and histograms, keeping the last charge and the greatest peak and intake, and keeping bundles apart. Only the core reads the accounts' counters, through the runner's gauges. A store stream keeps all its entries for one age.

## Decision

- **Counting.** The runner's watch of an incarnation counts its handler calls by run time in the 32 latency buckets, and its gauge gives them with the bundle, the incarnation and the abandonments of the instance so far.
- **Sampling.** A task of the core reads each instance's gauge every second, and holds a sample of each instance charged anything in that second in the backlog `journal` takes with the drains of its rings; an instance's energy is its CPU time at `PowerConfig.busy_mw`.
- **Resolutions.** `/journal/metrics` holds one stream per resolution, `seconds`, `minutes` and `hours`, kept 1 h, 24 h and `metrics_h` and sharing the metrics' share of `/journal`; `journal` appends each second's samples and the merged samples of each minute and hour as they close.

## Consequences

- A sample holds what the gauges count: no wakeups, I/O or network bytes per account.
- No `MachineSample` is taken: sampling the CPUs from a task of the core each second stalls the boot.
- The minute and the hour open when `journal` starts are merged from the samples it takes after it.
