# 0044. Lost notifications

Status: accepted

## Context

Chapter 11 has a subscription send `Changed` for each commit it watches and, once the subscriber's credit is spent, one `Lost` counting the commits it could not send. The store tells subscribers from its sequencer, which every request waits on, and a post through a port waits for credit.

## Decision

- A port's `offer` posts a one-way call only when the port has credit left, giving whether it did.
- The store offers each notification. A subscription counts the commits it could not tell of, and its next notification first offers one `Lost` with that count.

## Consequences

- A subscriber that misses commits learns of them at the next commit it watches.
