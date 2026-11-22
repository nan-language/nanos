# 0142. Recomputing admissions

Status: accepted

## Context

Chapter 09 recomputes the provenance class and `shared` field of every admitted bundle a change of the policy affects, and runs admission again for every bundle whose admission relied on a signature by a revoked key; chapter 10 then stops the refused bundle's instances, which enter `failed` with the refusal's code. An admission record keeps the sealers and reviewers the policy of its admission accepted, so the class cannot be recomputed from the record alone. `policy` checks a new record of `/sys/policy` on its own subscription, after the commit writing it; `keys` reads `/sys/principals` at each verification. `admission` starts lazily, and the runner had no order ending an instance in `failed` with a given fault.

## Decision

- **Trigger.** `supervisor` subscribes to `/sys/policy` and `/sys/principals` from its start on, and on each commit changing them posts `recompute` to `admission`, naming the commit when it changed the policy.
- **The policy's commit.** `Trust` carries the commit that wrote the record `policy` last checked; `recompute` reads the trust until it reflects the named commit, for at most 30 s.
- **Judgement.** `recompute` reads each admitted bundle but the base one, verifies its signatures with former keys allowed, and judges it again under the trust: a changed class or `shared` field is written to its record; a signature that no longer verifies refuses it.
- **Refusal.** Each instance whose specification or `/live` record names a refused bundle gets the `Control` order `Fail(code)`, which stops it and leaves it `failed` with that fault; then `/sys/code/<hash>` is deleted.

## Consequences

- A recomputation reads every admitted bundle blob whole.
- A policy or key change made while `supervisor` does not run is judged only at the next one.
- A recomputed class moves no instance, every instance running in the shared space.
- An instance of a refused bundle that starts again fails to load its image.
