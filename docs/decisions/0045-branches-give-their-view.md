# 0045. Branches give their View

Status: accepted

## Context

Chapter 11 has `branch` return the new branch's `Branch` record and a View of it. A NaN request's result cannot hold a capability beside other values (nan-language/nan#144).

## Decision

- `branch` returns the View alone; `/history/branches/<name>` holds the branch's record.
- The store holds the root entry of each branch's Views until `delete_branch` deletes it, which revokes them.

## Consequences

- A caller that needs the record reads `/history/branches/<name>`.
- A restart of `store` revokes the Views `branch` gave.
