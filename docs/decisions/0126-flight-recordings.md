# 0126. Flight recordings

Status: superseded by 0134

## Context

Chapter 17 arms an instance's flight recorder by `record`, during probation and after a trap, keeping a recording of its state at arming, its inputs and the results of its nondeterministic calls, committed to `/state/<moniker>/recording`, and replays a recorded input on forks. Only a holder sees a component's state, at quiescence, and a `sys::Stepper` gives the encoding of each message it handles through `Wire.step_encoded`; `nanos::Instance`, which runs bundle components, gives neither, and forks never run bundle components. A fork's clock and random source pass through a tape that records or serves back their reads. A store record holds at most 4 MiB. A replay counts as a preview of the requesting session, whose authority only the link holds.

## Decision

- **Kept by the driver, filled by the holder.** Arming is a holder order: the holder takes the base at quiescence and, while armed, steps through `step_encoded`, recording each message with the reads its tape took, a replayed call with the reads its preview recorded. Past its bound a recording takes a new base at the next quiescence, and each checkpoint becomes its base. The recording lives in the instance's driver, outside the domain.
- **Periods.** `record` arms for its duration. A probation arms for its span, and reaching its own bound ends it. A domain ending commits the recording and defers the rest of the period to the next start, from the state that start restores; a trap also defers 10 minutes, ended by the next checkpoint.
- **A blob.** The recording's `mem::wire` encoding is drafted in chunks and committed through the instance's View of `/state`, from a task of the driver's scope.
- **Re-run on forks.** `Control.rerun` runs the recording's inputs from its base up to the one asked, each on a fork from the state the previous fork ended with, serving each its recorded reads. A fork calling another instance diverges. A bundle other than the recorded one is refused as `not-forkable`.
- **Replay under the requester's authority.** `history --replay` runs `journal::replay_input` with the session's Control and View; the `replay` job runs it with its grants. Unless `with_head`, the forks' store reads reach a branch made at the commit that wrote the recording, deleted after. `journal` serves `record` only: a `replay` operation would act with `journal`'s authority for any caller.

## Consequences

- Bundle components are never recorded, so their probations record nothing and their reverts replay no input.
- A re-run's store reads see the store as the recording was committed, not as each input saw it.
- An input whose message holds a capability does not re-run.
