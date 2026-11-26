# 0164. Frames isolated jobs and instances lend, in spaces of their own

Status: accepted

## Context

NaN has `run_isolated` and `hold_isolated` lend an isolation the frames their job or instance draws from: the space's runtime bounds its draws by them, a draw past them ends it alone, and they come back once it ends (nan-language/nan#311). `isolate`'s frames serve the space itself, the image of each bundle it maps among them, which it keeps until it runs nothing. A task of the executor `isolate` takes runs the space's thread for every job and instance of the space, and the core gives a job's or instance's space the executor of its account, by which the scheduler shares the CPU and the runner charges its energy.

## Decision

- **A job or instance lends its memory limit.** The core lends the space of an isolated job or instance the frames of its memory limit and gives `isolate` the rest: the space's allowance and two copies of its bundle's image. A run of the `nan` job lends 768 MiB, its space keeping 128 MiB.
- **Spaces of their own.** The core still makes a space for each job and instance: one space shared by several would charge all of them to one account (N70), and could not take the frames of a bundle joining it later (N71).

## Consequences

- An isolated job or instance drawing past its memory limit ends `Trapped` with `trap.exhausted`, alone.
- Each isolated job and instance pays a space's tables, runtime and bundle copy.
- A kernel task relaying an isolated instance that its memory cannot serve halts the kernel while nan-language/nan#318 stands.
- Supersedes 0161's decision giving each job and instance a space of its own until N67, and its consequence that awaits nan-language/nan#314.
