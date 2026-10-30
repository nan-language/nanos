# 0007. Changes as previewed plans, gated by computed reversibility

Status: accepted

## Context

An agent acts fast, sometimes wrongly, and sometimes steered by injected text. Asking a human to approve every action produces fatigue that voids approvals. What matters is the small set of effects nothing can undo:

- data leaving the machine;
- physical effects;
- erasure;
- spending.

NaN reserves a digest field for reversibility. NaN code reaches the world only through capabilities, so running real code against substitutes is a faithful dry run.

## Decision

- Every operation has a reversibility class, `none < recorded < restorable < irreversible{emit, actuate, erase, spend}`. The checker computes it in the reserved digest field, protocols declare it per operation, and device operations seal it.
- A plan's preview forks the base commit. It runs the real handlers of the target instances, one level deep, with recording substitutes for capabilities of irreversible class. It returns an effect set and a plan hash.
- Apply re-validates everything the preview read and replays the recorded inputs, so the result equals the preview or the plan fails. An irreversible effect is released only with the previewed bytes.
- A plan commits in spans, each ending with at most one irreversible step. Each irreversible effect is written to the outbox in its span's commit and delivered after that commit and its health window, at least once, under the request key.
- Activation is health-checked: a failure reverts the commit that caused it. Changes that can cut the agent's link are provisional until confirmed.
- Policy and human approval attach to points of no return, widenings of authority, trust escapes, and changes to the boot chain, retention and policy. Everything else proceeds within the actor's grants and can be undone.

Alternatives rejected:

- **Approval per action:** fatigue.
- **Static analysis alone for previews:** imprecise where running the code is exact. The static class bounds the preview but does not replace it.
- **Re-executing live at apply:** an input changed between preview and approval would release an effect the human never saw.
- **Two-phase commit across components:** blocking and heavy. Spans with effects at their end, and sealed compensations, cover what agents meet.

## Consequences

- An agent may change almost anything, and the system names exactly what cannot be undone.
- Previews cost a fork of the targets' states, bounded to 1 MiB per plan and charged to the session.
- Instances holding hardware are not forked: their steps take their protocol's declared class.
- The digest field and the limit words that bound it need [N16](../spec/22-nan.md#n16). Substitute capabilities outside tests need [N27](../spec/22-nan.md#n27).
