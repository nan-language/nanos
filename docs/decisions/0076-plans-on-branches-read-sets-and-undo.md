# 0076. Plans on other branches, read sets and undo

Status: accepted

## Context

Chapter 14 lets a plan commit to a branch other than the running branch with `--at <branch>`. Such a plan holds only `write`, `delete`, `install`, `grant`, `revoke` and `revert` steps, its preview forks the head of that branch, and its spans commit there. Each span validates the plan's read set before it pauses its targets and replays its steps, and fails with `plan.stale` when a node changed. `undo` of a plan reverts its span commits and the commits carrying its cause.

A View names one branch, and the store mints Views of a branch only for the request that creates it. A handler cannot wait, so it reaches the store only through keyed tasks, and a fork's keyed tasks never run. A commit that changes no versioned node is unrecorded, and the store drops its metadata at the next index flush. `link` opens no cause per request.

## Decision

- **Views of existing branches.** The store gives a View with admin rights a View of an existing branch with its subtree, rights, principal, instance or session and origin labels, refusing preview and job branches. It holds the root entries of these Views and of acting Views per holder and branch, the least recently used of 64 giving way.
- **Plans on other branches.** `--at` names the branch a plan previews from and commits to, `main` by default. A step its branch cannot hold fails the plan's preview with `plan.step-misplaced`. Previews, applies, confirmations and cancellations act on the plan's branch, and `get --at <branch>` reads its head. A write there needs a grant of that branch or of every branch. Its changes activate no instance. At its start, `link` also recovers the plans of every branch `/history/branches` lists, other than preview and job branches.
- **Read sets.** A forked run reads no node, so a plan's read set is the nodes its tree steps write and those its `--expect` flags name, and it records no `/live` read. Each span reads its nodes again through the plan's acting View before it pauses its targets. Then its commit validates them atomically.
- **Undo.** The inverse of a plan reverts the span commits of the spans whose steps change a node of the previewed delta. A plan with no such span has no inverse.

## Consequences

- Only instances holding Views with admin rights create branches; no link verb does.
- A stale plan fails before it replays a step, so its instances' states stay unchanged.
- Until `link` opens a cause per request, `undo` of a plan reverts no commit that its released messages and keyed tasks make.
