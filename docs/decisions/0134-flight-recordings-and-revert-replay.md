# 0134. Flight recordings and revert replay

Status: accepted

## Context

Chapter 17 arms an instance's flight recorder by `record`, during probation and after a trap, keeping a recording of its state at arming, its inputs and the results of its nondeterministic calls, committed to `/state/<moniker>/recording`, and replays a recorded input on forks. Chapter 10 reverts an update by running the old code from the pre-update encoding on each input the new code handled during probation, holding their effects, a trapping input becoming a dead letter. Only a holder sees a component's state, at quiescence. A `sys::Stepper` gives the encoding of each message it handles through `Wire.step_encoded`, and a `nanos::Instance`, which runs bundle components, through `Instance.step_encoded`; each lets its holder take the messages a step sent the component and drop the keyed tasks it holds. A bundle component's holder takes no orders while it waits for a message, and forks never run bundle components. A fork's clock and random source pass through a tape that records or serves back their reads. A store record holds at most 4 MiB. A replay counts as a preview of the requesting session, whose authority only the link holds.

## Decision

- **Kept by the driver, filled by the holder.** A component of the kernel image is armed by a holder order: its holder takes the base at quiescence. A bundle's component is armed by an order left in the recorder, which its holder carries out before its next step, taking the base from its checkpoint. While armed, a holder steps through `step_encoded`, recording each message with the reads its tape took, a replayed call with the reads its preview recorded. Past its bound a recording takes a new base at the next quiescence, and each checkpoint becomes its base, except during a probation, whose recording keeps every input since the update. The recording lives in the instance's driver, outside the domain.
- **Periods.** `record` arms for its duration. A probation arms for its span, and reaching its own bound ends it. A domain ending commits the recording and defers the rest of the period to the next start, from the state that start restores; a trap also defers 10 minutes, ended by the next checkpoint.
- **Revert replay.** The driver keeps the inputs of a probation's recording when it ends. A revert restarts the old code from the pre-update encoding, which handles each of them before it resumes, its holder dropping what the step sent the component and the keyed tasks it holds. A trap during the replay makes the input a dead letter and restarts the replay without it; `update.reverted` is recorded once the replay ends, with the inputs replayed and dead-lettered.
- **A blob.** The recording's `mem::wire` encoding is drafted in chunks and committed through the instance's View of `/state`, from a task of the driver's scope.
- **Re-run on forks.** `Control.rerun` runs the recording's inputs from its base up to the one asked, each on a fork from the state the previous fork ended with, serving each its recorded reads. A fork calling another instance diverges. A bundle other than the recorded one is refused as `not-forkable`.
- **Replay under the requester's authority.** `history --replay` runs `journal::replay_input` with the session's Control and View; the `replay` job runs it with its grants. Unless `with_head`, the forks' store reads reach a branch made at the commit that wrote the recording, deleted after. `journal` serves `record` only: a `replay` operation would act with `journal`'s authority for any caller.

## Consequences

- A bundle component armed while it waits for a message takes its base when the next one arrives.
- Recordings of bundle components do not re-run, as forks never run them.
- A re-run's store reads see the store as the recording was committed, not as each input saw it.
- An input whose message holds a capability does not re-run.
- A revert replays only inputs whose handler returned: the input whose trap reverts an update becomes a dead letter.
