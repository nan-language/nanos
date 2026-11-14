# 0094. Transfer accounting

Status: accepted

## Context

Chapter 08 has the driver performing a transfer charge its bytes to the account of the message that requested it; chapter 13's `Account` reads them as `io_bytes`, and chapter 17's metric samples split them into `io_read_bytes` and `io_write_bytes`. A block driver queues each request from a handler, which cannot wait, and answers it from the handler of its completion, which runs under the driver's own header. A volume view forwards each request from a keyed task, which carries the header of the call it forwards. The core keeps what is folded into each account by number beside the gauges.

## Decision

- **Meter.** The core mints a `Meter` capability for the block drivers. A driver keeps with each queued request the account its header names, and charges the request's bytes, read or written, through the meter once the device completes it without error; the meter offers each charge to a queue of 256 that a core task folds into the account.
- **Reading.** A transfer folds into its account's bytes read or written, a session's account only while it is open; an instance's gauge reads those folded since its incarnation started, a session's those since it opened. `Account.io_bytes` is their sum, and a metric sample holds each one's growth over its second.

## Consequences

- A charge offered while the queue is full is dropped, as is a failed transfer.
- The store's transfers for a request it handles at once are charged to the request's account, those of its own work to the store.
