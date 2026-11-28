# 0172. `boot` halts through the power switch and mirrors each generation's base bundle

Status: accepted

## Context

When the reboot at the health gate's deadline fails, the kernel halts; `boot` runs the gate, and only the core halts. The preview of a generation switch checks the new state schemas against the base bundles of the generations the switch falls back to, which only their genesis records on the boot volume name; the link reads the tree, not the boot volume.

## Decision

- **`boot` holds the power switch.** Its Env carries the switch the core gives `power`. Once `power` answers the gate's reboot, which it does only when the reboot failed, `boot` orders `Halt` with its generation and the core halts with `fault.power-failed`.
- **`/sys/boot` names each generation's base bundle.** `boot` reads each generation's genesis record from the boot volume once, checked against its manifest's blob id, and mirrors its base bundle's code hash. The link reads those bundles' admission records in `/sys/code`, which genesis keeps for every base bundle that ran.

## Consequences

- `boot` could order any switch, not only `Halt`.
- A fallback generation whose base bundle's admission record was removed goes unchecked.
