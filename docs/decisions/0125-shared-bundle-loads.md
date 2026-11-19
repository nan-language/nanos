# 0125. Shared bundle loads

Status: accepted

## Context

Chapter 10 loads a bundle into a space when an instance of it starts there and no other instance uses it, shares its code and read-only frames, and unloads it when none uses it, emitting `code.loaded` and `code.unloaded`. The core keeps the loader and loads each image an instance's or a job's domain reads through its own cell. Until isolated spaces exist, every bundle loads into the shared space.

## Decision

- **One load per code hash.** The runner passes each start's code hash with its image; the core keeps one load of each bundle, counts the jobs and instances using it, and unloads it when the last ends.
- **Events from the core.** The first load records `code.loaded` with the image's bytes, the last unload `code.unloaded`, both naming the shared space.
- **The rest of bundle components.** A specification naming another bundle than the base bundle is created from the component its admission record describes; the first task of each start's domain reads `/sys/code/<hash>/image`, checked against the blob id admission recorded, through the instance's cell; the kernel holds the instance with `nanos::hold`, which serves it as a component of the kernel image is served, refusing holder orders but health probes; `describe` gives the protocol schema the loaded bundle gives for it.

## Consequences

- Each start of a loaded bundle still reads its image before the core finds the bundle loaded.
- A plan whose preview needs a fork of a bundle's instance cannot preview it.
- A trap in the bundle's code abandons the instance's domain only.
