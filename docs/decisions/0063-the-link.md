# 0063. The link

Status: accepted

## Context

Chapter 18 has `link` serve one protocol on every transport: request lines gathered over continuation lines and cut into words, flags and forms; blocks of a verdict line, records and the verdict line again, cut to a budget in units with an elision line; records whose fields render canonically as chapter 02 defines it; sessions opened by `hello`, in `dev` mode by `hello owner` on serial at once; requests not ended within 100 ms answering `pending`; and a primer shipped in the kernel image. The serial transport is the UART of the `console` role, which the sandbox binds to its standard streams. A NaN component's handlers never wait, so its I/O runs in keyed tasks that post messages back; derived `Render` writes construction text, whose forms are the canonical rendering's but for the identifiers chapter 02 renders in their text forms. A stored record carries only the canonical path of its type and its schema hash, and the running store registers no schema trees. A generic type's associated function taken as a function value is `generic.uninferred` even with explicit type arguments.

## Decision

- **Package.** `link` holds the request grammar, blocks, the canonical rendering of the base bundle's records, sessions, the primer and the `link` component, a base instance the supervisor starts at step 9 with a read View of `/`, the console role's `read` and `write`, `keys`' `verify`, the kernel's entropy and the clocks.
- **Rendering.** `ident` renders each identifier in its text form through `Render`, and every record type of the base bundle derives `Render`, so its derived rendering is canonical. `link` renders a record through a table from the canonical path of its type to its typed decoding and rendering; a type the table does not hold renders as its path and size.
- **Serial.** A keyed task reads the UART, cuts its bytes into lines ending in LF, CR or CRLF, gathers them into requests through a validator and posts each request, refusal and console line; another writes the blocks waiting, one at a time. The ready record goes out before any block once the link has read `/live/status`, the events stream's cursor, `/live/security` and `/live/boot`.
- **Requests.** The component refuses a request the session rules refuse, answers `hello` itself, and runs every other request in a keyed task of its own that reads through the link's View and posts its block, a timer answering `pending` past `LinkConfig.pending_ms`.
- **Sessions.** `hello <principal>` draws a nonce and answers a challenge binding the serial connection to the link's identity key when `/sys/principals` gives the principal a key, and `keys` verifies the signature of its `HelloProof`.
- **Streams.** A stream's node gives the sequence number its next entry takes, so the link writes cursors and a stream's first and last records without reading it.

Alternatives rejected:

- **Rendering from schema trees:** they name no types, so nested records, enums and identifiers would render without their names.
- **A registry of renderers:** each entry would need a function value per type, which explicit type arguments cannot give.
- **Reading the store from the component's handlers:** handlers cannot wait.

## Consequences

- A type the base bundle stores renders canonically only once the link's table names it.
- A request reads through the link's View after the session rules; capabilities minted per session come with the requests that write.
- Lines ending in a lone CR, as a terminal sends them, end requests too.
