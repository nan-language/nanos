# 0165. Capabilities in calls of isolated instances

Status: accepted

## Context

NaN gives the operations of a crossing kind the handles of the capabilities their arguments and results pass beside their bytes, and carries no call with handles over a bridge (nan-language/nan#312). A call an isolated instance makes through a port or a store View goes up its link to the core through the capability's crossing kind, which puts it in the intake of the target's runner ([0162](0162-calls-of-isolated-instances-through-their-runners.md)).

## Decision

A call of an isolated instance through a port or a store View carries its capability arguments as the handles of the crossing call: the kind puts them, after the View itself for a View, in the call it hands the target's runner, and the reply's capabilities come back down as the handles of its answer. The kinds whose operations pass no capability, `Live`, the journal's sources and the start of `keys`, refuse a call carrying handles.

## Consequences

- Supersedes 0162's consequence that a call or reply carrying a capability fails as malformed until N68.
