# 0037. The rendered tree

Status: accepted

## Context

Chapter 13 has `/history` render the store's commit metadata, branch records and pins, and `/live` hold observed state that its providers publish and nothing stores: a read returns the last record published, never waits for an instance and costs no wake-up, and a write below either fails with `store.denied`. The runner holds each instance's lifecycle, the core its measured facts, and an instance publishes its summary through `Cx.publish`, a method of its runner capability that cannot wait. The drivers of the runner wait on the store while they restore checkpoints, and a spin lock's body cannot allocate.

## Decision

- **History.** The engine renders `/history/commits/<n>` as the `CommitMeta` of each retained commit, `/history/branches/<name>` as the `Branch` record of each branch, its name taking a segment per segment, and `/history/pins/<n>` as the pins of each pinned commit, listed in numeric and byte order. A View reaching `/history` reaches every retained commit.
- **Board.** A task of the core holds the latest record published at each path below `/live`, `/live/machine` from the start, and answers reads, providers and readers posting to it over one channel. It computes `/live/status` when it is read, from the records of instances and of the store, the boot and the clocks.
- **Live.** `store` serves `/live` through a `Live` capability on the board, which also publishes the store's record at `/live/store` after its mount and each run of commits.
- **Instances.** Each driver publishes its instance's `LiveInstance` whenever its lifecycle, stop, restarts, last fault or summary changes, and withdraws it when the instance is deleted. `Cx.publish` hands the summary to the driver through the queue the replies to postponed requests take. The reconcile state follows the lifecycle and the specification's stop.

Alternatives rejected:

- **The board under a spin lock:** its records allocate.
- **Providers computing their records at each read:** a read would wait on drivers that wait on the store.
- **The runner publishing `/live/status`:** its uptime and time would age between publications.

## Consequences

- A `LiveInstance` names no memory, CPU time or desired commit, its intake as of its last change, and `shared` as its space.
- `/live/devices`, `/live/cpus`, `/live/memory`, `/live/accounts`, `/live/security`, `/live/boot`, `/live/sessions`, `/live/plans` and `/live/net` have no provider yet.
- `/live/status` counts no approvals or exposures and names cursor 0; `/live/machine` names the store's storage `File`, and no IOMMU, watchdog or NIC.
- `count` and `scan` reach only stored dirs.
