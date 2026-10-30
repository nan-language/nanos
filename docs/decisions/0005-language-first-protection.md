# 0005. Protection by the language first, hardware by computed placement

Status: accepted

## Context

NaN safe code has no undefined behaviour and holds authority only as values. Running all checked code in one address space therefore removes system calls, copies and TLB flushes between components. The language leaves five threats unanswered:

- a compiler or prover defect reached by adversarial source;
- a false premise of trusted code;
- speculative reads across parties sharing an address space;
- DMA from devices;
- code no trusted principal reviewed.

NaN 1 has no user mode, and today any kernel trap halts the machine.

## Decision

- Two kinds of address space:
  - the **shared space**, ring 0, the kernel's identity map;
  - **isolated spaces**, ring 3 or EL0, one per principal and provenance class.
- Placement is computed at admission from facts, never chosen by code. A bundle runs shared only when all of these hold:
  - it is vendor or reviewed code, or agent code under the autonomous preset on a machine with an IOMMU;
  - every trust escape carries a signed seal the policy accepts;
  - it holds no key-use capability;
  - it acts for `system` or the owner.
- The shared space is one confidentiality domain. Secrets live only in isolated spaces and in `keys`.
- All agent-written code has provenance `agent`, whatever its session's taint, because one model drives many sessions. Only a human review promotes it.
- With an IOMMU, each device is confined to its own domain. Without one, the machine is one trust domain for DMA: drivers need human or vendor seals, and agent-written drivers are refused.

Alternatives rejected:

- **One address space for everything:** speculation and toolchain defects would reach every secret and every principal.
- **A hardware-isolated microkernel for every component:** every crossing would pay 0.3–2 µs and a copy, for integrity NaN already gives code of one principal.
- **Provenance from session taint:** it is laundered by opening a new session.
- **Protection keys or software fault isolation:** too few keys, and absent on most AArch64 machines.

## Consequences

- Reviewed code pays no protection cost on its hot paths. Unreviewed code pays what a process pays today.
- On a machine without user mode, bundles placed isolated are refused, and `keys` runs shared and is reported unconfined ([09](../spec/09-authority.md)).
- Isolated spaces depend on NaN user mode ([N23](../spec/22-nan.md#n23)), and confined drivers on an IOMMU model ([N13](../spec/22-nan.md#n13)).
- Placement states its cost (split tables, IBPB, core scheduling) where it is decided.
