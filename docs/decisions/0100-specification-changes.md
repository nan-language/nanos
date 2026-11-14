# 0100. Specification changes

Status: accepted

## Context

Chapter 06's reconciliation table maps each change of a running instance's specification to its effect: a restart for its principal, component, placement or reserved budget, the next start for its memory limit and laziness, the next quiescence for its share and other fields. `supervisor` reconciles `/sys/instances` with starts, stops and creations. NaN changes an account's share while its tasks run.

## Decision

- **Order.** `Control.specify` gives the runner an instance's specification. Each reconciling pass of `supervisor` gives it every specification, creating the instance only when the runner holds none, then starts or stops it.
- **Effects.** The runner restarts an instance that runs, starts or hibernates when the change touches the fields the table maps to a restart. A changed share of a running incarnation ends its CPU reservation, admits the new share's, and has its watch reshare its account at the next quiescence. The runner keeps every other field for where it reads it next: its memory limit, durability and handler limits at the next start, its restart policy at the next abandonment.
- **Previews.** A change of a running instance's specification that restarts it shows as a `Restart` activation.

## Consequences

- A running instance whose new share's reservation fits on no CPU runs without one until its next start, which fails with `budget.over-reservation`.
- The runner receives each instance's specification at every pass, and compares it with the one it holds.
