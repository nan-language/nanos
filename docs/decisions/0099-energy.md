# 0099. Energy

Status: accepted

## Context

Chapter 08 charges each account the energy of its CPU time at the CPU's power estimate and gives an account with an energy budget a bucket holding at most an hour of budget, refilled continuously: an empty bucket throttles the account, whose entities run as `background` until it refills to a minute of budget, and `/live/accounts` shows each account's energy and state. The accounts of a principal's sessions drain one bucket. NaN gives an account the CPU time its tasks ran and changes its executor's share while its tasks run, the policy seeing the new share from its next call.

## Decision

- **Charging.** The watch of each incarnation charges its account's new CPU time at the power of a running CPU after each handler call and at each reading of its gauge, which carries the energy used and whether the account is throttled. A session's gauge charges its own and its principal's bucket at each reading.
- **Buckets.** The watch keeps the account's bucket in nanojoules: full at the start, refilled by the time since it last filled at the budget per hour, capped at the hour's budget, and drained by the energy charged. An empty bucket marks the account throttled until it holds a minute of budget.
- **Throttling.** When the bucket's state changes, the watch has the account's executor run in the `background` class with its weight, or in its own share again; an incarnation's watch does so at quiescence.

Alternatives rejected:

- **Charging at each reading only:** an account read seldom would drain its bucket in one step, missing the refills in between.
- **A throttled flag the policy reads per account:** every scheduling decision would look the account up, where a share change costs one call per change of state.

## Consequences

- An instance's account changes class only between handler calls, and a session's when its gauge is read, at least once a second while metrics sample it.
- The bucket of an instance restarts full with each incarnation.
