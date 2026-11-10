# 0051. The journal

Status: accepted

## Context

Chapter 17 has `journal`, the only reader of the core rings, append each event to `/journal/events` as an `Event` naming its source instance, its kind from the kernel image's kind table, its severity, cause and principal, and its payload, which the chapter defining the kind fixes. Genesis declares the streams of `/journal`, each keeping its entries for the hours its configuration gives and within its share of the part of the store's capacity `/journal` holds. Chapter 09 builds an instance's Env with its configuration record, decoded as its component's configuration type. A ring record names its source by its slot in the runner's table and carries 8 bytes of payload, each continuation 32 more. The core prints each event it drains on the serial console, the sandbox's only view of events, and `store` and the drivers it waits on start before `journal`.

## Decision

- **Package.** `journal` holds the event records and their payloads, the kind table, the declarations of the streams and the `journal` component. The kernel's `rings` module writes ring records by the kind table's indices.
- **Feed.** The core drains the rings and prints each event as before, then holds the records, each with the moniker its slot names, in a backlog of at most 2,048 records, counting the records of a drain past it as lost. `journal` takes the backlog through a `Feed` capability the core mints for it.
- **Appends.** `journal` appends what each take gives to `/journal/events` in one commit whose entries derived encoders wrote: each event but the `debug` ones unless `keep_debug`, and one `journal.lost` for each ring that lost events. At each start it declares its streams again with what its configuration and the store's capacity keep.
- **Slots.** The runner tells its journal the moniker each slot names as the slot's driver starts.
- **Configuration records.** The runner reads `/sys/config/<moniker>` at each start of an instance whose component takes a configuration record, through a View of `/sys/config` of its own, and hands it to the domain, where the Env decodes it: a missing record leaves the component's defaults, and one that does not decode fails the start.
- **Trap kinds.** The continuation of an `instance.trapped` record holds the trap's kind.

Alternatives rejected:

- **Reading every instance's configuration record at its start:** the drivers `store` waits on would wait on `store`.
- **`journal` reading the rings itself:** the console would show no event while `journal` does not run.

## Consequences

- An event's payload holds what its ring record carries: `store.mounted` names the bytes replayed alone, a device event the bus path and the argument.
- Every event acts for `system` under the boot's system cause.
- The core pool holds up to 2,048 drained records while `journal` does not run.
- The kernel's boot heap holds the tasks of `journal` beside those of the other instances only once NaN lets a kernel run more tasks than its 16 MiB boot heap fits ([nan-language/nan#147](https://github.com/nan-language/nan/issues/147)); until then the sandbox exhausts it at boot.
