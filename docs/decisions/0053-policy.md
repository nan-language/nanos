# 0053. Policy

Status: accepted

## Context

Chapter 19 has `policy` decide the subjects of plans and `call`s by the owner-signed record of `/sys/policy`, which it accepts at each boot and each write only once it is valid: signed by the owner key or the genesis record's policy, of a version above the accepted one and for the machine id, its patterns parsing, no approver, reviewer or sealer glob reaching an agent, and no `autonomous` preset or agent code in the shared space without `iommu`; an invalid record makes it decide by `locked`. Genesis writes the `supervised` preset; `policy` draws the machine id at its first start after genesis into `/sys/config/policy`. The `keys` instance verifies signatures, and a store subscription tells a port of the subscriber of each commit below its prefix.

## Decision

- **Package.** `policy` holds the records of chapter 19's policy, the presets as code, the patterns with their parsing and matching, the evaluation of subjects, and the `policy` component.
- **Protocol.** `Oversight` serves `decide` and `explain` from the policy `policy` holds, and the store's notices of `/sys/policy` through a port wired to `policy` itself.
- **Genesis policy.** Genesis writes `supervised`, version 1, for the zero machine id and unsigned; a record is the genesis record's policy when its encoding is that one's.
- **Machine id.** Started without a machine id in its configuration, `policy` draws one and writes it as its configuration record, then checks the record of `/sys/policy` with it.
- **Checks.** At each start and each notice, a keyed task reads the record, checks its validity in chapter 19's order, verifying a signature through `keys`, and hands the outcome to a handler, which decides by the policy from then on and emits `policy.changed`, or emits `policy.invalid` and decides by the policy it accepted before, `locked` when it accepted none.
- **Unmatched conditions.** A rule's `first_use` holds for every subject.

Alternatives rejected:

- **Reading the genesis policy from the image's genesis record:** the record's policy is the preset, which `policy` builds itself.

## Consequences

- `ask`, `approve`, `withdraw` and `propose`, approvals, agent budgets, taint and the rule of two are not served; a subject carries its taint, budget overruns and rule of two as its caller computes them.
- The first start after genesis writes `/sys/config/policy`, which `supervisor` answers by restarting `policy`.
- A record unsigned in `dev` mode passes the signature check.
