# 0009. Human oversight by signatures, and containment of injected text

Status: accepted

## Context

Humans reach NaN OS through the AI, so oversight cannot rest on a channel the AI controls. Agents read hostile text, and any of it may carry instructions. Containment must therefore come from structure, not from the model's obedience. NaN seals bind a principal to an item hash and lapse on any change, but language 1 seals are unsigned.

## Decision

- **The owner.** Each machine has one owner, claimed at the console with physical presence. The owner's public key is kept in TPM NV or a boot-services-only UEFI variable, and the owner signs the policy.
- **The policy.** It is data: rules matching principals, operations or reversibility classes, objects and conditions to allow, notify, ask or deny. The default preset `supervised` asks for exactly the triggers of 0007.
- **Approvals.** An approval is an Ed25519 signature by a human principal other than the requester, over the approval, the plan hash and the read-set hash. It may travel over any channel, since only the signature counts. For points of no return, the policy may require a channel independent of the requesting session. Unanswered approvals expire into a denial.
- **Taint.** A session's taint grows only when it receives enveloped content. The rule of two asks for approval when a plan combines a tainted session, a sensitive read and a point of no return.
- **Extraction.** The only untainted path from hostile content is extraction into parsing types, in a quarantined child session, within 64 bits per origin per hour.
- **Audit.** The audit chain is hash-chained and anchored by signatures that the owner's client fetches. Emergency controls revoke every agent session at once.

Alternatives rejected:

- **Approvals as console prompts only:** a headless machine would have no oversight.
- **Information-flow labels on every message and object:** they creep until everything carries every origin, and cost a lookup on hot paths.
- **Policy as code:** unbounded evaluation, hard to review, and another injection channel.
- **Trusting session freshness as cleanliness:** a model carries context across sessions.

## Consequences

- No agent can approve its own plan, and an approval covers exactly the effects the human saw.
- Without a TPM the audit key is held in software, and the chain is tamper-evident only off the machine.
- Signed seals and approvals that a device verifies need [N15](../spec/22-nan.md#n15).
