# 0087. Specification changes

Status: accepted

## Context

Chapter 06's reconciliation table maps each change of a running instance's specification to its effect: a restart for its principal, component, placement or reserved budget, the next start for its memory limit and laziness, the next quiescence for its share and other fields. The runner held each instance's specification from its creation, and `supervisor` reconciled `/sys/instances` only with starts, stops and creations, so no change reached an instance the runner already held. NaN fixes an account's share at its split (nan-language/nan#220).

## Decision

- **Order.** `Control.specify` gives the runner an instance's specification. Each reconciling pass of `supervisor` gives it every specification, creating the instance only when the runner holds none, then starts or stops it.
- **Effects.** The runner restarts an instance that runs, starts or hibernates when the change touches the fields the table maps to a restart; it keeps every other field for where it reads it next: its share, memory limit, durability and handler limits at the next start, its restart policy at the next abandonment.
- **Previews.** A change of a running instance's specification that restarts it shows as a `Restart` activation.

## Consequences

- A changed share takes effect at the instance's next start, until an account's share can change (N42).
- The runner receives each instance's specification at every pass, and compares it with the one it holds.
