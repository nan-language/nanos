# 0093. Session consumption budgets

Status: accepted

## Context

Chapter 18 fails a session's requests with `budget.exhausted`, naming the budget, once its principal's sessions together spent the `cpu_ms` or `energy_mj` of their budget. The core counts each session's CPU time and energy on its account, which closes with the session; `link` holds the budgets and its per-principal tallies of output for its lifetime, and opens the session's account through the core before running each request's work. NaN offers no limit on an account's CPU time.

## Decision

- **Consumption.** The core keeps what the closed sessions of each principal consumed, read from their gauges as they close, until it closes every session's account at `link`'s start. Each account it opens or gives again comes with its principal's consumption: that of its closed sessions with what the gauges of its open ones read.
- **Refusal.** `link` fails a request with `budget.exhausted` when the consumption given with its session's account reached the budget's `cpu_ms`, then its `energy_mj`.

## Consequences

- Work admitted before a budget is spent runs to its end, so the sessions can pass their budget by the work of the requests already running.
- A restart of `link` starts its principals' consumption afresh, as it does their output.
