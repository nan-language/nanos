# 0029. Runtime suspend through the runner

Status: accepted

## Context

Chapter 15 suspends a device under `PowerPolicy.Auto` once its driver has had no request for its protocol's idle time, 10 s for `Entropy`: the runner calls the driver's `suspend()`, `devices` puts the device in D3hot, and a request or the policy `On` wakes it, `wake()` coming before the request. Only the runner sees an instance's messages, and only the holder of a component's stepper calls `suspend()` and `wake()`, between handlers. A holder waits for messages through `Stepper.incoming()` alone: NaN lends the stepper only to a scope's spawns, so waiting for another event as well costs a task per wait, and cancelling the holder's scope could stop it between the marks the watchdog reads. A driver holds its function in its Env while it runs, and `PciFunction.set_power` keeps the function's configuration across D3hot. A function without a power management capability has no D3hot.

## Decision

- **The setting.** `Control.suspend_after(moniker, ms)` gives an instance the time without a message after which it is suspended, `None` for never; the runner keeps it across incarnations and delivers it to the running domain. `Control.suspended(moniker, seen)` answers once the instance's suspension differs from `seen`.
- **The holder.** While the setting is set, the holder waits for a message within that time; when none comes, it calls `suspend()` and reports it, then waits without a deadline and calls `wake()` before handling the next message, reporting that too. A new setting holds from the holder's next wait.
- **The device.** The driver's `suspend()` quiets the device and puts its function in D3hot; `wake()` turns it on and sets the device up again. `virtio-rng` resets the device first, failing the requests still waiting with `device.reset`.
- **`devices`.** The keeper lists a function's power management capability. Under `Auto`, `devices` orders an entropy driver whose device has one to suspend after 10 s, and otherwise never; it reseeds the core's random source from the driver after each order, a request through which the order holds at once and a suspended driver wakes. It follows each entropy driver's suspension, recording the device's power and emitting `device.suspended` and `device.resumed`.

Alternatives rejected:

- **A task per wait racing the holder's `incoming()` with a power order:** a task spawned for every message an instance gets.
- **Cancelling the holder's scope to deliver an order:** the holder could stop between the marks of a handler, which the watchdog would take for a handler past its limit.
- **`devices` turning the function off through the keeper:** the function is in the driver's Env while the driver runs.
- **Suspending a device without power management:** it would be recorded suspended while it stays in D0.

## Consequences

- Messages, completions included, restart an instance's idle time, so a driver relaying its last interrupt after a quiet second suspends 1 s later than its last request plus the idle time.
- Each wait of an instance whose setting is set arms a timer; a suspended instance arms none.
- A request still waiting when its driver suspends fails with `device.reset`.
