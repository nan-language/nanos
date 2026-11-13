# 0081. CPU reservations

Status: accepted

## Context

Chapter 08 admits a `latency` share's reservation, `runtime_us` in every `period_us`, when its instance starts, on the CPU whose admitted reservations use the smallest fraction of it, the lowest numbered on ties, provided they then use at most 80% of it; a reservation fitting on no CPU is refused with `budget.over-reservation`, and it ends when its instance stops. The `latency` work of the account then runs only on that CPU. A kernel's policy sees a task's share, whose `reserve_ppm` holds the reservation's ratio, but not the CPU it was admitted on nor its period, and cannot tell which account an executor serves (nan-language/nan#198).

## Decision

- **Admission.** The runner keeps the millionths of each CPU its admitted reservations use. A start of an instance whose share is `latency` admits its reservation unless it holds one, and fails with `budget.over-reservation` when the least used CPU cannot take it within 80%.
- **End.** The reservation ends with the incarnation's domain, except at a hibernation, and when the start fails after admitting it; the next start admits it again.

Alternatives rejected:

- **Encoding the CPU in the share's class:** the policy would bind the work, but the period would stay unknown and the class would mean two things.

## Consequences

- A reservation admitted on a CPU does not yet bind its account's `latency` work there, nor renew its runtime each period.
- A plan's preview does not report the CPU a reservation gets.
