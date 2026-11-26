# 0163. Hibernation of bundles' instances

Status: accepted

## Context

The holder of an instance of a bundle's component, `keys` among them, refused every order of its driver but health probes, so an order to hibernate one was never answered. That holder serves the instance through NaN's `nanos::Instance`, which only the task holding it reaches. It waits for the instance's messages through `incoming`, which no operation wakes without a message.

## Decision

- **Ordered hibernation.** The dispatcher of a bundle's instance flags an order to hibernate it. Its holder looks for the flag before each wait and at least every second while it waits, then makes the instance's image: it calls the handlers of the messages that wait, keeps a checkpoint when the state is dirty, encodes the state and takes the messages the instance sent itself that wait for their delay. It hands the driver the replies queued and the image, then ends the instance. A revival resumes it with the reason `Revived`.
- **Pressure.** A hibernation only an idle instance takes, which pressure orders, leaves a bundle's instance serving.

Alternatives rejected:

- **Racing `incoming` against a signal of the dispatcher:** each side of a race runs on a task of its own, while the instance is reached only by the task holding it.

## Consequences

- The holder of an idle bundle's instance wakes once a second.
- Pressure frees no memory of a bundle's instance.
- A bundle's instance still refuses its driver's orders to pause, replay, arm its flight recorder and copy its state.
