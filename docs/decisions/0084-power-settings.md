# 0084. Power settings in the core

Status: accepted

## Context

Chapter 08's `PowerConfig`, at `/sys/config/power`, configures rules the core applies: the power of a running CPU that energy is charged at, the energy-performance preference of each class and the idle states. `power` holds the record. It is lazy, starting at its first message, and the runner reads its record into its Env at each start. A change of the record restarts it, and `supervisor`, subscribed since genesis, replays at each boot every change since genesis. The core has no View of the store and no subscription of its own.

## Decision

- **Path.** When the kernel builds `power`'s Env, it applies the record the runner read for that start, or the defaults without one, before `power` runs: `power`'s switch carries the settings to the core's power task, which writes them into the scheduler's state.
- **Restarts.** A restart starts a lazy instance at once, as chapter 06's lifecycle table moves a `hibernated` instance to `starting` on a restart. A change of the record therefore takes effect when it commits, and a changed record takes effect at boot, when `supervisor` replays its change.
- **Energy.** The runner charges energy at the power the machine keeps, read at each charge, and the core's metrics take each sample's energy from what the runner charged.

Alternatives rejected:

- **The core reading the record itself:** it would need a View and a subscription that `power`'s Env and `supervisor`'s replay already provide.

## Consequences

- `power` runs once its record has changed, idle.
