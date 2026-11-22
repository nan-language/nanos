# 0146. Load-time registries keyed by name

Status: accepted

## Context

Chapter 10 makes the kernel image's registries load-time registries: loading a bundle adds its contributions, admission step 11 refuses a contribution whose key a live entry has, and an `install` preview lists the bindings its `drivers` entries cause. Chapter 15 lets an entry bind a device by itself only when its bundle's provenance is `vendor` or `reviewed`, and reruns matching when a contributing bundle loads or unloads. The only registry the kernel image declares is `drivers`; no kernel policy registry exists yet.

NaN 0.26 publishes a bundle's entries to a loaded registry at `Prepared.load`, reads them without publishing through `Prepared.entries`, refuses a duplicate key at load, and retires a placement without publishing anything through `Prepared.retire`; a retired placement is freed only by `Loader.reclaim`. A loaded registry's key may be a `text::Fixed[N]` field, which loading orders as the text it holds. The core alone holds the loader; base instances run in the shared space, so they read the core's tables directly.

## Decision

- **Entries.** A `drivers` entry is keyed by its name, a fixed text of 20 bytes.
- **Previews and holds.** The core's loading task serves the `Registries` capability beside the loads of jobs and instances: a preview places the image, reads its entries and reports the first whose name a live or earlier entry has; it keeps the last four placements for the load that follows. A hold loads the bundle for its contributions until released; a placement dropped unloaded is retired. After each order the task reclaims the placements nothing uses any more.
- **Admission.** Step 11 runs after provenance: admission has the core preview the image it takes, refuses a duplicate with `code.registry-duplicate`, records each entry as a contribution to `devices::drivers`, and asks `devices` which drivers the entries would start, its reply to `admit` carrying them for the preview to list as `Start` activations.
- **Holding.** `supervisor` watches `/sys/code`: each pass holds the admitted bundles whose records list `drivers` contributions, releases the others, and gives `devices` the sources of the loaded entries (bundle and whether it binds by itself, from the record's provenance) when they change, with the commit the drivers it starts converge to.
- **Matching.** `devices` reads the registry for every entry, the sources telling a bundle's from the kernel image's, and on new sources reruns its devices left unbound and those whose entry a source names.

## Consequences

- A preview of a bundle never installed keeps its placement until four newer ones replace it.
- Admission refuses `drivers` entries only; a kernel policy registry needs its own step.
- A provenance recomputed in a record reaches `devices` at the next pass.
