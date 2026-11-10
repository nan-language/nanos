# 0054. Boot, clock and power

Status: accepted

## Context

Chapter 20 adds three base instances: `boot`, which records each boot in the boot record, `/sys/boot` and `/live/boot`; `clock`, which keeps wall time from the RTC and the readings `set` gives; and `power`, whose `off` and `reboot` are `restorable` because the core, not the instance, writes the power registers. `clock` and `power` are lazy. The std package `boot` is the alias a kernel's entry takes, and NaN charges a hardware write to the reversibility of the handler that makes it.

## Decision

- **Packages.** `generations` holds the `boot` component, `clock` the `clock` component and `power` the `power` component with the switch capability. `generations` is named for what the instance keeps, the boot record's generations, since `boot` would shadow the std alias in every package depending on it.
- **Recording.** At each start, a keyed task of `boot` reads both copies of the boot record through `fat`, writes the boot counter into the current copy when the loader did not, compares the record with the previous `/sys/boot` to find a boot that ended before its health gate, commits `/sys/boot` and publishes `/live/boot`. Without a boot record, as in the sandbox, `/sys/boot` holds what the handoff record tells.
- **Wall time.** `clock` starts from the core's RTC reading at its start, keeps wall time as monotonic time plus an offset that slews or steps, and writes the RTC through a port to the `rtc` role.
- **Power.** The core mints a `Switch` whose orders a task of the core carries out; `power` orders the switch from a keyed task, so its handlers stay `restorable` and hold no hardware. The core writes the sandbox's exit port.

## Consequences

- Off and reboot reach only the sandbox: the FADT's registers, `\_S5` and PSCI, the intake pause, the checkpoints, the index flush and the watchdog of chapter 08 are not carried out.
- `clock` gives no `WallClock` capability and reads no network time source; the instances read the core's wall clock of its start.
- `boot` runs no health gate and serves no generation operation; `/live/boot` gives 0 for the kernel's entry counter and frequency.
