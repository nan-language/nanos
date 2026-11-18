# 0120. Bundle components

Status: accepted

## Context

Chapter 10 runs instances of the components a component bundle exports. NaN's `nanos::hold` holds such an instance in a domain of its own from a loaded bundle, its checkpoint and a runner capability, and gives the kernel a `nanos::Instance` taking encoded calls and the steps, checkpoints, health probes, suspensions and end a holder asks for, but not the component's state type; `sys::load::Load.schema` gives the protocol schema of each component entry of a loaded bundle. The runner drives the components of the kernel image through typed steppers, which also serve forks, replays, hibernation and the orders a holder carries out between handler calls. A port copies the protocols of its target when it is minted, so the runner must know an instance's protocol hash when it creates the instance. Admission records each component's state schema, protocol hash, operations and Env fields.

## Decision

- **Description at creation.** A specification naming another bundle than the base bundle, whose code hash the runner's configuration holds, is created from the component its bundle's admission record describes, which the runner reads from `/sys/code/<hash>/admission` before it lists the instance; a code hash without an admitted component is refused as invalid.
- **Image at each start.** The first task of each start's domain reads `/sys/code/<hash>/image`, checked against the blob id admission recorded, through the instance's cell; the kernel loads it through the core task that loads jobs' images, holds the instance with `nanos::hold`, and unloads the image when the instance ends. A bundle that cannot be read or loaded fails the instance with the read's or the load's code.
- **A holder of its own.** The task of the instance's domain resumes it, probes its health, serves the calls its driver delivers through its inbox one at a time, steps it, suspends it after the time without a message the driver sets, keeps its checkpoints under the schema admission recorded, and reports to the driver as a component of the kernel image does. It refuses the orders a holder carries out between handler calls; forks, replays and hibernation are not offered for it.
- **Described by the bundle.** The kernel hands the holder the protocol schema the loaded bundle gives for the component, which the holder tells the driver as the incarnation starts; `describe` gives it when its hash is the instance's protocol hash.

## Consequences

- Each instance loads its own copy of its bundle's image, as each job does, placed in the boot heap.
- A plan whose preview needs a fork of a bundle's instance cannot preview it, and the link's `call` reaches a bundle's instance once an incarnation of it has started.
- A trap in the bundle's code abandons the instance's domain only, and reaches the driver as the trap that ended the incarnation.
