# 0080. Energy

Status: accepted

## Context

Chapter 08 charges each account the energy of its CPU time at the CPU's power estimate, `PowerConfig.busy_mw` without RAPL or activity monitors, and gives an account with an energy budget a bucket holding at most an hour of budget, refilled continuously: an empty bucket throttles the account to `background` until it refills to a minute of budget, and `/live/accounts` shows each account's energy and state. NaN gives an account the CPU time its tasks ran, but no way to change its executor's share once split, nor to name its executor to the policy (nan-language/nan#198). The sandbox's CPUs have no RAPL. The core does not read `/sys/config/power`; the metrics estimate energy at the default `busy_mw`.

## Decision

- **Charging.** The watch of each incarnation charges its account's new CPU time at the default `busy_mw` after each handler call and at each reading of its gauge, which carries the energy used and whether the account is throttled.
- **Buckets.** The watch keeps the account's bucket in nanojoules: full at the start, refilled by the time since it last filled at the budget per hour, capped at the hour's budget, and drained by the energy charged. An empty bucket marks the account throttled until it holds a minute of budget.

Alternatives rejected:

- **Charging at each reading only:** an account read seldom would drain its bucket in one step, missing the refills in between.

## Consequences

- A throttled account still runs in its class: nothing changes the share of its executor.
- A `busy_mw` configured in `/sys/config/power` does not change the estimate.
- The bucket of an instance restarts full with each incarnation.
