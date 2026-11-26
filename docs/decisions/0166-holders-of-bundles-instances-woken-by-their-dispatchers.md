# 0166. Holders of bundles' instances woken by their dispatchers

Status: accepted

## Context

NaN leaves 64 KiB of an isolated instance's memory drawn each time the holder of that instance cancels its wait for the instance's messages (nan-language/nan#317). The holder of a bundle's instance cancelled one such wait a second to look for the orders its dispatcher flags ([0163](0163-hibernation-of-bundles-instances.md)), so `keys` ran out of memory every three minutes. Only the task holding the instance may wait for its messages, but a task of a scope that task opens may cancel that wait.

## Decision

The holder waits for the instance's messages in a scope whose other task waits for its dispatcher's signal or for the holder's next deadline, the checkpoint falling due or the suspension of an idle instance, and cancels the scope when either comes. The dispatcher posts the signal with each order it flags: a hibernation, a health probe, a new suspension time.

## Consequences

- An idle bundle's instance costs nothing until a message, an order or a deadline comes.
- Each order or deadline that ends a wait of an isolated instance leaves 64 KiB of its memory drawn while nan-language/nan#317 stands.
- Supersedes 0163's decision that the holder looks for the flag at least every second, and its consequence that the holder of an idle bundle's instance wakes once a second.
