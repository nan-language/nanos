# 0136. Cancelling jobs

Status: accepted

## Context

Chapter 18's `cancel` cancels a job with acknowledgement, its result then `Cancelled(cancel)`, and classifies its operand after the session's pending requests and watches: a target holding a `.` or naming an instance is a moniker. A job acts for the principal of the plan that ran it, a principal that neither `JobSpec`, `/state/<moniker>/job` nor `/live/instances` records; the runner holds it in the job's instance specification. The link's `Control` reaches every instance, so the link must not cancel a job for any session.

## Decision

- **A `Control` order.** `Cancel` names the principal it cancels for. The runner refuses it as invalid for an instance that runs no job or whose job has ended, and as denied unless the principal is the one the job acts for, `owner` or `system`; otherwise it cancels the run with the reason `cancel` and answers once the run has ended.
- **The link classifies by the tree.** Once no pending request or watch of the session is the operand, a moniker naming an instance of `/live/instances` or `/sys/instances` is cancelled for the session's principal; a request id naming nothing keeps `request.unknown-name`.

## Consequences

- A session cancels only the jobs of its own principal, unless it is the owner's.
- An instance that is no job is `request.not-cancellable`.
