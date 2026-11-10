# 0041. The store asks its placement

Status: accepted

## Context

Chapter 13 has `/live/store` name the store's placement, its partition's bus path or `file`, and the store partitions left unused. `devices` places `store-volume` and alone knows which partition or image its view covers; `store` reaches its volume only through the role's `Block` port.

## Decision

- `devices` serves `placement`, giving the path below `/live/devices` of the partition `store-volume` lies on, `file` for the store image, and the paths of the other store partitions.
- `store` is wired to `devices` and asks it at mount.

## Consequences

- A placement that changes after the store mounts shows only at its next mount.
