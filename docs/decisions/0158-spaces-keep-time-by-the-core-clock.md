# 0158. Isolated spaces keep time by the core's clock

Status: accepted

## Context

NaN's `isolate` serves a space the clock its `Services` hold: its runtime keeps time by it, and its job's or instance's world holds it as its clock. An instance without a `Clock` grant holds a revoked clock, and a space whose clock fails takes every timer as due at once, so the task driving it never yields its CPU, starving the core's other work there, a block driver's completions included (nan-language/nan#304).

## Decision

The core's `Spaces` capability carries the core's monotonic clock, and every space's services hold it in place of the world's clock.

## Consequences

- An instance in an isolated space reads the monotonic clock without a `Clock` grant (N65).
