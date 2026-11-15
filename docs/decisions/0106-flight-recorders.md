# 0106. Flight recorders

Status: accepted

## Context

Chapter 17 has each instance's flight recorder hold the headers of its last 16 inputs, through its incarnations: number, time, cause, sender, operation, the XXH3-64 and size of its encoding, run time, flags and outcome; `history <moniker>` prints them. NaN gives a holder the header and the operation's number of each message a step handles, once the step returns; a step that traps returns nothing. A step does not tell a call the dispatcher posted from a message the component sent itself or a keyed task posted, and a handler's sends carry the header of the message it handles. The dispatcher posts one encoded call at a time, under the header of its handling.

## Decision

- **Holder.** The driver holds the recorder across incarnations: 16 headers by number modulo 16, how many inputs it took, and the header of the handler call running, which counts among the 16. The holders of its incarnations write it under a spin lock; a trap of the domain records the running header as trapped.
- **Attribution.** Before posting a call, the dispatcher sets aside its header with the XXH3-64, size and flags of its encoding and its operation's number, which the running header starts from. The first step whose header equals the one set aside takes them; any other step records a moved input of its header's cause and sender.
- **System interface calls.** The holder records its calls of `resume`, `health`, `pressure`, `suspend` and `wake` as inputs of the core, flagged as system interface calls and numbered 0 to 4.
- **Reading.** Each incarnation gives its driver the names of its component's operations. A `Control` order of the read right gives the inputs with the monikers of their senders' slots and their operations' names; `link` reads them for a session that reads the instance's specification.

Alternatives rejected:

- **Recording each delivered call at the driver:** messages a component sends itself, keyed tasks' posts and system interface calls would go unrecorded.
- **Setting the posted calls apart by a mark in their header:** the handler's own sends would carry the mark too, and a header's other bits are the specification's.

## Consequences

- A message whose header equals that of the call posted after it, taken first, is recorded as that call.
- The input of a handler call that traps records the call posted then, whichever message trapped, and no cause, sender or operation when none was posted.
- Each encoded call costs an XXH3-64 of its encoding, and each handler call two acquisitions of the recorder's lock.
