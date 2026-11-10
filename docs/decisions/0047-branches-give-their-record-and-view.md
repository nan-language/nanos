# 0047. Branches give their record and View

Status: accepted

## Context

Chapter 11 has `branch` return the new branch's `Branch` record and a View of it with the requesting View's subtree and rights. NaN lets a handle record, a struct of encodable and capability fields, be an operation's result. The Views the store mints derive from root entries the store holds.

## Decision

- `branch` returns `Branched { branch, view }`.
- The store holds the root entry of each branch's Views until `delete_branch` deletes the branch, which revokes them.

## Consequences

- A restart of `store` revokes the Views `branch` gave.
