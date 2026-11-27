# 0167. One isolated space per principal and class

Status: accepted

## Context

NaN charges the time a space's thread runs each job or instance to the executor the kernel places it with (nan-language/nan#315). It draws each bundle's image from the frames of the job or instance whose start maps it, and unmaps the image after the last of them (nan-language/nan#316). It runs the kernel's work for each job or instance in a domain of its own (nan-language/nan#318). So one isolation can hold the jobs and instances of a principal and provenance class while each keeps its own account, memory and faults. A holder that cancels its wait for an instance's messages no longer leaves the instance's memory drawn (nan-language/nan#317). A space's label is cut to 32 bytes for events, so two keys can share a label.

## Decision

- **One space per key.** The core keeps one isolation per space key: the principal and provenance class, or the moniker of an instance alone. The isolation lasts from the start of the key's first job or instance to the end of its last. The whole key, not the label, tells spaces apart.
- **Each job or instance brings its own.** It lends the space the frames of its memory limit and of its bundle's image, runs on its account's executor and reaches its world's services. The space keeps 16 MiB of its own. A run of the `nan` job lends 832 MiB, 64 MiB of them for the toolchain's image, and its space keeps 64 MiB.
- **A bundle isolated once per space.** The core isolates a bundle once for each space it runs in and shares it among its jobs and instances there. It records `code.loaded` as the first of them comes and `code.unloaded` as the last leaves.
- **The space's own services deny.** The space keeps time by the core's clock. Its log, standard streams, random source and file system deny every operation, so that no world reaches another's through them.

## Consequences

- A space's tables and runtime are paid once per principal and class.
- A job or instance whose bundle its space already maps lends frames for an image it does not draw, and its budget gains them.
- A job or instance that never waits holds up the others of its space, running only as its own share lets it.
- Supersedes 0164, and 0166's consequence that awaits nan-language/nan#317.
