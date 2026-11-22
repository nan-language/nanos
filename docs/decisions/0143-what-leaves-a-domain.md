# 0143. What leaves a domain

Status: accepted

## Context

NaN carves each allocation of up to 64 KiB from the chunk its domain's pool carves for the allocation's cell, and keeps 8,320 bytes of boot heap for each chunk it carves. A value moving into another domain's reach, as a message to the core or to another instance does, takes the chunks holding its memory with it: they leave the sender's pool, their free space is never carved again, and they return only once every allocation in them is freed. What the sender keeps in such a chunk keeps it drawn and charged to the sender's cell, and once the sender's domain ends nothing frees it.

For each commit, about one a second on an idle machine, `store` answered the request, called its volume and posted `/live/store` from its own cell, each send taking along the chunk where it had just carved what it keeps of the commit. A chunk stayed behind per commit: the store's cell grew until it trapped `exhausted` at its 64 MiB limit, the trapped incarnation's chunks stayed drawn for good, and the boot heap holding their records filled 27 minutes after boot, halting the kernel.

## Decision

- An incarnation hands its driver its replies and summaries copied through a cell split from its own for its life, which holds nothing else.
- `store` draws what it sends out of its domain through a cell split from its own for the life of its sequencer: its calls to its volume, `devices` and `keys`, its posts and reads of `/live`, its subscriptions' calls and its grants.

Alternatives rejected:

- **A larger boot heap:** it delays the halt, while the store still traps and leaves its chunks drawn.

## Consequences

- Only what a domain keeps stays in its cell's chunks: on an idle machine `store` levels off at about 14 MiB and the boot heap stays flat.
- An instance whose state grows with its traffic draws what it sends through a cell of its own, as `store` does.
- What the copies take counts toward the instance's cell while they travel.
