# 0144. Cleaning as the log grows

Status: accepted

## Context

The store cleaned only when a write lacked room and at `gc`, each time reading every record of a whole 8 MiB segment into memory, so the log grew until the volume filled. Once free segments fell below the reserve of 2 plus 0.5%, `free_bytes` still offered writes the open segment's room: writes drew the reserve down to nothing, and every write lacking room ran thinning, cleaning and three index flushes even when none could free anything. Index flushes then failed with `store.full`. Writing a node had withdrawn it from memory before appending its record, and a log finding no segment to follow its successor had already sealed an end-of-segment batch, so committed nodes were lost and the batches after became unreplayable.

## Decision

- **Steps.** After each run of requests, while commits have grown the log since the latest index flush, the sequencer has the engine clean one step: at least 1 MiB of a full segment holding under a quarter of live data, read a batch at a time and resumed at the next step, its copies made durable at the step's end. While free segments, those freed awaiting their index flushes counted, number at most 4 plus 1% of all segments, it steps until no candidate is left.
- **Batches.** A batch holding 1 MiB takes no further record, so a step holds at most one batch beside its copies.
- **The reserve.** Writes get no room once free segments fall below the reserve. A write lacking room has cleaning and index flushes run only when thinning thinned commits, a segment is being cleaned or awaits its flushes, a candidate exists or the log grew since the latest index flush.
- **Failures.** A node stays in memory until its record is appended, and a log with no segment to follow its successor changes nothing, so an index flush failing with `store.full` loses nothing and runs again later.
- **Replay.** Replay passes over a commit record numbered below the next commit, the cleaner's copy of one it holds; the segments it reads through are full, and the one it stops in ends with an empty batch.

## Consequences

- An idle store never cleans; a busy one reads about a megabyte of segment per run of requests while candidates exist.
- A segment being cleaned stays full until its last step, and becomes free two index flushes later.
- Data copied together with live data into one segment stays there while that segment holds a quarter or more of live data; thinning, not cleaning, frees such segments.
