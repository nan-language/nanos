# 0170. The generation switch through `boot`, the store's running branch and the TCO watchdog

Status: accepted

## Context

A generation switch installs a generation on the boot volume, reboots into it on a trial branch, and lets the health gate decide it. The store reads blobs by path, not by blob id. Every instance reaches `main` through Views it was granted at its start, and the core holds the grants of `main` as its live grants. The plan step `generation` must fail with `boot`'s code when the install fails, and a reboot loses every reply after it. QEMU's q35 has the ICH9's TCO timer, which resets the machine at its second expiry once its NO_REBOOT bit is clear; the scheduler's hooks run with interrupts disabled, and the core's deferrable work never wakes an idle machine.

## Decision

- **`boot` copies from the tree.** `install` and `replace_loader` read their blobs through a View reading `/data`: the manifest at the path the operation names and the generation's other files beside it, each checked against the manifest's blob ids, then copied to the boot volume a piece at a time through `fat` and read back. Operations run one at a time on a keyed task; `rollback` and `recovery` reboot through `power` once their reply is given.
- **The store holds the running branch.** At mount the store creates or keeps `trial/<n>` from the boot it is told of and redirects every View of `main` to it, so no instance's grants change; the core's live grants follow the running branch. `take_trial` and `drop_trials`, for a View writing `/sys/boot`, which only `boot` holds, take the branch into `main` or delete those below the floor; an owner's merge of the trial branch into `main` takes it too.
- **`boot` runs the health gate.** It polls the store, `/live/instances` and `link`'s published summary every second, and owns the deadline's reboot. Once the gate passes it gives the store the least `store_format` of the generations the boot record names, and the store upgrades a volume of the format before its own when that least format is its own.
- **The `generation` step is a command.** Its preview reads the new genesis record beside the manifest: the running state schema of each checkpointed base instance must be the new component's or one of its recorded earlier schemas, a base component an instance of another bundle has a port to must keep its protocol hash, and every running bundle needs an image for the new support routines unless the generation ships the `nan` job. Its apply calls `boot`'s `install`, ends the plan with the failure's code, then sends `power` a one-way `reboot`; the step forks no instance.
- **The TCO timer as the watchdog.** The start sequence finds the ICH9's LPC bridge, enables its root complex registers at their conventional address when the firmware left them off, and clears NO_REBOOT; deferrable core work refreshes the timer, the scheduler's `idle` hook stops it when the machine becomes idle and its `slice` hook arms it again.

## Consequences

- A generation's files must be uploaded beside its manifest.
- While a trial runs, no View reaches `main`; the owner merges the trial branch to take it.
- A `boot` that stops running gives up the gate's deadline.
- A state schema the new base bundle reads only through a migration it does not record as earlier refuses the switch.
- The i6300esb the sandbox also carries stays unused.
