# Answers — Chapter 14: Assembly, Concurrency, and Reader APIs

[Return to the lesson](../chapters/14-assembly-and-concurrency.md)

## 1. Thread-role split

Retriever and drain are long-lived coordinators. They perform operations that
can block: ranged I/O, queue publication/acquisition, and waiting for another
stage. Virtual threads make one pair per projected column practical and
unmount during those waits.

Page decode is short-lived, CPU-bound, and high-frequency. It runs on the one
bounded platform-thread pool owned by `HardwoodContext`. A virtual thread per
page would move CPU work onto process-wide carrier scheduling without an
explicit decode-parallelism bound and would add needless task/thread
overhead.

## 2. Slot reuse for sequence 9

```text
9 % 8 = 1
```

Sequence 9 may use slot 1 only after sequence 1, its prior occupant, has been
drained. The retriever's invariant

```text
nextSeq - consumePosition < 8
```

must hold before submission. When `nextSeq == 9`, this requires
`consumePosition >= 2`, proving sequences 0 and 1 have been consumed and slot
1 is free.

## 3. Sequence 4 completes before sequence 3

The decode task for sequence 4 stores its result in slot `4 % N` and unparks
drain. Drain still probes only slot `consumePosition % N`, which represents
sequence 3. If that slot is null, it parks; it does not skip ahead to 4.

When sequence 3 completes, its decode task stores into the expected slot and
unparks drain. Drain consumes 3, increments position, and can then consume the
already-ready 4. Completion is parallel; assembly remains ordered.

## 4. Flat copy versus nested inspection

For a flat column, page position `i` is top-level record `i`, so a kept
contiguous record interval is the same interval in the typed values array.
`System.arraycopy` is sufficient, with a parallel validity update.

For nested data, one top-level record can occupy several level/value
positions. `NestedColumnWorker` uses repetition level 0 to find record starts,
apply masks in record coordinates, and avoid splitting records across
batches. A blind leaf-position arraycopy would count lists as extra rows and
misapply masks.

## 5. Why page-boundary flushes break alignment

Parquet permits each column to choose independent page boundaries. Column A
might end a page after row 100 while column B ends one after row 140. If each
worker publishes on page end, their first batches contain different row
counts even though each column's values are individually correct.

A row reader would then join values from different row positions or reject
the count mismatch. Hardwood batches therefore cut on common top-level record
capacity and coordinated boundaries, not arbitrary page ends.

## 6. Full back-pressure chain

When the consumer does not poll:

1. two completed batches fill `readyQueue`;
2. drain blocks in timed `publish` offers;
3. drain no longer removes reorder slots, so `consumePosition` stops;
4. decode completions fill up to `MAX_INFLIGHT_PAGES` slots;
5. retriever observes `nextSeq - consumePosition >= MAX_INFLIGHT_PAGES` and
   parks;
6. parked retriever no longer calls `PageSource.next()`;
7. no more decode tasks are submitted and no later demand-driven chunks are
   requested.

Recycling mode can also block waiting for the consumer to return a free batch
holder, with the same upstream effect.

## 7. `ColumnReader` exchange mode

`ColumnReader` uses detaching mode. Each batch is factory-allocated because
public accessors promise that arrays from earlier batches are never reused or
overwritten and may be retained or handed to another thread after
`nextBatch()`.

Production remains bounded by the ready queue's capacity of two and by the
worker's bounded reorder buffer. Detaching grants ownership of consumed
arrays; it does not permit an unbounded number of unpublished/pending
batches.

## 8. Why close awaits decode futures

Stopping retriever prevents new submissions, and stopping drain prevents new
assembly, but an already admitted decode task may still be decompressing or
reading a `ByteBuffer` slice. That slice can be backed by a memory mapping,
direct buffer, or remote-backend resource owned by the parent input.

Worker close joins the coordinators and waits for all admitted decode futures
so parent close can safely release those resources. Releasing them first can
cause use-after-close failures, corrupted reads, or native memory faults.
