# Chapter 14 — Assembly, Concurrency, and Reader APIs

**Study time:** 45–75 minutes

Decoded pages are still column-local storage units. Hardwood must preserve
their order, combine them into bounded batches, align projected columns,
convert nested levels into navigable structures, and stop producing when the
consumer stops consuming. This chapter covers that concurrent hand-off and
the ownership contracts exposed by row and column readers.

## Objectives

By the end of this lesson, you should be able to:

1. trace the retriever, decode-task, and drain roles of `ColumnWorker`;
2. explain how the circular reorder buffer preserves page order while allowing
   parallel completion;
3. contrast flat and nested assembly, including their batch-boundary rules;
4. follow back-pressure from a stalled consumer to stopped page retrieval;
5. explain recycling versus detaching `BatchExchange` ownership and connect
   each mode to `RowReader` or `ColumnReader`;
6. identify shutdown and error-propagation invariants that prevent resource
   use after close.

## Prerequisite recap

- `PageSource.next()` yields `PageInfo` in file/row-group/page order for one
  projected column.
- `PageDecoder` produces typed `Page` objects. Decode work is CPU-bound and
  pages can take different times depending on codec, encoding, and size.
- Flat pages have one top-level row per page position. Nested pages can have
  multiple level positions per record; repetition level 0 begins a top-level
  record.
- A `PageRowMask` selects page-relative top-level record intervals.
- Batches are an in-memory Hardwood unit. Parquet files do not contain
  “Hardwood batches.”

## Central mental model: ordered hand-off through bounded stages

The authoritative pipeline is:

```text
RowGroupIterator -> PageSource -> ColumnWorker -> BatchExchange -> consumer
     shared          per col       per col         per col
```

Inside one `ColumnWorker`:

```text
Retriever virtual thread
  PageSource.next()
  create/update decoder for the returned page
  park while nextSeq - consumePosition >= N
  assign sequence number
  submit short decode task ------------------------------+
                                                          v
Shared bounded platform-thread decode pool        decode PageInfo
                                                  store DecodedPage
                                                  in slot seq % N
                                                  unpark drain
                                                          |
                                                          v
Drain virtual thread
  read only consumePosition % N
  assemble in sequence order
  publish bounded batch
  consumePosition++
  unpark retriever
```

There are two kinds of concurrency:

- **Coordination:** two long-lived virtual threads per column. Blocking I/O,
  queue waits, and `park()` are suitable here.
- **CPU:** short page-decode tasks on the one bounded platform-thread executor
  owned by `HardwoodContext`.

Decode does not run on virtual-thread carrier threads. The shared executor is
the explicit limit and balancing point for CPU-bound work across columns.

## Worked example: out-of-order decode, in-order assembly

Let `MAX_INFLIGHT_PAGES = 3` for illustration. The actual default is 8.
Retriever assigns:

```text
sequence 0 -> slot 0
sequence 1 -> slot 1
sequence 2 -> slot 2
```

Suppose decode completion order is page 1, page 2, page 0:

```text
after page 1: [null, page1, null]
after page 2: [null, page1, page2]
after page 0: [page0, page1, page2]
consumePosition = 0
```

The drain always probes `consumePosition % N`, so it parks after the first two
completions: slot 0 is not ready. When page 0 arrives, it atomically removes
page 0, assembles it, increments `consumePosition`, then immediately consumes
pages 1 and 2.

This is a reorder buffer, not a work queue. Sequence is implicit in
`consumePosition`; slots are safe to reuse only because the retriever enforces:

```text
nextSeq - consumePosition < MAX_INFLIGHT_PAGES
```

Each slot also owns reusable `PageDecoder.LevelScratch`, a file name, and a
row-group `filterAlwaysMatches` flag. Retriever writes the plain arrays before
the decode task publishes to the `AtomicReferenceArray`. The volatile
publication/read chain makes those fields visible to the drain. Moving slot
reuse or position increments would require re-proving that happens-before
relationship.

At end of input, retriever waits for a free slot, stores an
`EMPTY_SENTINEL`, and unparks drain. The sentinel appears after all assigned
sequence numbers, so drain publishes its partial batch only after every prior
page.

## Flat assembly

`FlatColumnWorker` copies kept page ranges into fixed-capacity batch storage:

```text
Page.IntPage.values() --System.arraycopy--> Batch.values int[]
definition levels    --presence mapping---> Batch.validity long[]
```

For a mask with intervals `[0, 4)` and `[7, 9)`, it calls the same range-copy
path twice. That path:

1. limits the copy to remaining batch capacity;
2. limits it to remaining `maxRows`, when applicable;
3. copies physical arrays;
4. records present bits;
5. publishes when `rowsInCurrentBatch == batchCapacity`;
6. continues the same page in a fresh batch when needed.

Validity uses set-bit-means-present semantics. A `null` validity reference is
the compact all-present representation. On the first absent value, the worker
backfills present bits for earlier positions and then maintains the bitmap.
At publish, a non-null bitmap is copied to exactly
`(recordCount + 63) >>> 6` words.

Variable-width byte arrays are appended into `BinaryBatchValues`: concatenated
bytes plus sentinel-suffixed offsets. Fixed-width physical types stay in
primitive arrays.

## Nested assembly

`NestedColumnWorker` cannot equate leaf positions with records. It maintains:

```text
nestedValues
nestedDefLevels
nestedRepLevels
nestedRecordOffsets
nestedValueCount
rowsInCurrentBatch
```

In the regular path, repetition level 0 begins a top-level record. A record is
never split merely because a page ends. Likewise, a batch must not be flushed
at an arbitrary page boundary: sibling columns generally have different page
boundaries and would become row-misaligned. Batches cut at common top-level
record capacity, file boundaries, row limits, and uniformly configured filter
boundaries.

For page positions:

```text
rep:    [0, 1, 1, 0, 0, 1]
values: [a, b, c, d, e, f]
```

the top-level records are:

```text
record 0 -> positions 0..2
record 1 -> position 3
record 2 -> positions 4..5
```

The row mask is interpreted in this record coordinate space, not leaf-position
space.

Before publishing, the worker prepares the shape needed by its consumer:

- `ALL_ITEMS` for `NestedRowReader`: raw levels plus all-items element
  validity and layer offsets;
- `REAL_VIEW` for unfiltered `ColumnReader`: real-items-only layer view and
  compacted leaf values, avoiding a consumer-thread scan;
- `REAL_VIEW_KEEP_LEVELS` for exact-filtered `ColumnReader`: retain raw levels
  so selected records can be compacted and the real view rebuilt.

The fixed-list path can use arithmetic boundaries and omit explicit levels
while its strict shape remains valid. If a batch must mix that page with a
regular/different-width page, the worker materializes the omitted levels
inside the same batch instead of flushing at a misaligned page boundary.

## BatchExchange and ownership

Every exchange has a `readyQueue` of capacity 2. Its mode determines how the
drain gets the next holder.

### Recycling mode: row readers

`BatchExchange.recycling` preallocates three holders:

```text
one held by drain + up to two in readyQueue
```

The consumer returns a finished batch to `freeQueue`. `FlatRowReader` and
`NestedRowReader` consume this way. Their row APIs are mutable cursors/views:
internal arrays can be reused after a batch is returned because callers read
values through the current row, not by retaining internal batch arrays.

### Detaching mode: column readers

`BatchExchange.detaching` calls its factory for each batch. There is no
`freeQueue`; the consumer permanently owns each result. `ColumnReader` uses
this mode because its public accessors return arrays directly.

The contract is stronger than “valid until next call”:

```text
batch 0 array != batch 1 array
advancing never overwrites batch 0
caller may hand batch 0 array to another thread
```

The `ColumnReader` cursor itself remains single-consumer-threaded. Safe array
hand-off does not make `nextBatch()` concurrently callable.

Both modes preserve bounded production because `readyQueue` has capacity 2.
Detaching prevents aliasing, not unbounded queued batches.

## Back-pressure, step by step

If a consumer stops polling:

```text
1. readyQueue fills (two published batches)
2. drain's exchange.publish waits in timed offers
3. drain stops consuming reorder slots
4. consumePosition stops advancing
5. completed pages occupy the bounded reorder buffer
6. retriever may already call PageSource.next() once for the next PageInfo
7. retriever sees nextSeq - consumePosition >= MAX_INFLIGHT_PAGES and parks
8. that PageInfo is not assigned or submitted until capacity returns
9. while parked, no further PageSource calls or decode submissions occur
```

The source-before-throttle order means step 6 can resolve one page and may
initiate its demand-driven chunk read even though the decode window is full.
The bound is on admitted decode tasks/reorder slots, not on possessing one
pending `PageInfo`.

In recycling mode, failure to recycle can additionally block the drain while
it waits for a free holder. The same pressure still propagates upstream.

When the consumer resumes, drain publishes/takes a holder, consumes pages,
advances `consumePosition`, and unparks retriever. A fast column therefore
self-limits while a slower column can retain decode-pool demand. The shared
pool naturally gives resources to work that remains queued.

Timed queue operations are not busy waiting. They allow threads to observe
`finished` and exit during close/error without relying on an extra poison
element that might itself need queue capacity.

## Reader construction and lifecycle ownership

Public creation selects assembly and exchange behavior:

```text
ParquetFileReader
  |
  +-- rowReader()
  |     schema flat   -> FlatRowReader
  |     schema nested -> NestedRowReader
  |     one recycling exchange + worker per projected column
  |
  +-- columnReader(...)
        flat   -> FlatColumnWorker   + detaching exchange
        nested -> NestedColumnWorker + detaching exchange
```

`FlatRowReader` owns hot primitive-array/validity references for all projected
columns and moves one row index. `NestedRowReader` uses drain-computed nested
indexes. `ColumnReaders` coordinates multiple `ColumnReader` instances and
validates equal record counts.

Closing any child reader stops its own worker:

- marks/finishes workers;
- unparks and joins retriever/drain virtual threads;
- waits for all admitted decode futures;
- does not close parent-owned `InputFile`s.

Iterator ownership differs by public entry point. An exclusively owned,
single `ColumnReader` closes and unregisters its iterator. `RowReader` and
`ColumnReaders` share an iterator across sibling workers, so closing those
children leaves that iterator tracked until the parent reader closes.

Waiting for decode futures is essential: a decoder may still hold a mapped or
direct page buffer. The parent `ParquetFileReader` may close input resources
only after children are quiescent. Repeated close is guarded and safe.

## Parquet format rules and Hardwood choices

### Format rules

- Page order within a column chunk is meaningful.
- Definition and repetition levels preserve null/empty structure and record
  boundaries.
- Different columns may use different page boundaries and page sizes.
- Repetition level 0 identifies a new top-level record in repeated paths.
- Parquet does not define Hardwood's in-memory batch size, thread model,
  validity bitmap representation, or array ownership API.

### Hardwood choices

- One retriever and one drain virtual thread coordinate each projected column.
- Every column submits page decoding to a shared bounded platform-thread pool.
- An `AtomicReferenceArray<DecodedPage>` of default size 8 reorders completion
  without boxed map keys/nodes.
- Slot-owned level scratch arrays reduce allocation and are protected by the
  same in-flight bound.
- Flat batches use primitive arrays and set-bit-present packed `long[]`
  validity.
- Nested assembly computes expensive index/view structures on the drain side.
- A bounded `BatchExchange` provides both visibility and back-pressure.
- Row readers recycle three holders; column readers detach fresh arrays.
- File transitions force a batch flush so one batch has one attributable file
  name.
- Shutdown waits for both coordinators and all decode tasks before input
  resources can be released.

## Current code tour

1. [`ColumnWorker`](../../core/src/main/java/dev/hardwood/internal/reader/ColumnWorker.java)
   defines the retriever/drain lifecycle, sequence slots, throttle, sentinel,
   error path, and quiescent close. Read `runRetriever`, `decode`,
   `runDrain`, `drainReadyPages`, and `finishDrain`.
2. [`FlatColumnWorker`](../../core/src/main/java/dev/hardwood/internal/reader/FlatColumnWorker.java)
   copies page ranges, constructs packed validity, handles binary buffers, and
   optionally evaluates a drain-side column matcher.
3. [`NestedColumnWorker`](../../core/src/main/java/dev/hardwood/internal/reader/NestedColumnWorker.java)
   detects records, handles masks, owns growing accumulators, and chooses an
   index mode.
4. [`NestedBatch`](../../core/src/main/java/dev/hardwood/internal/reader/NestedBatch.java)
   is the nested hand-off structure.
5. [`NestedLevelComputer`](../../core/src/main/java/dev/hardwood/internal/reader/NestedLevelComputer.java)
   derives validity, offsets, and real-items-only views.
6. [`BatchExchange`](../../core/src/main/java/dev/hardwood/internal/reader/BatchExchange.java)
   implements recycling/detaching modes and the finish/error polling protocol.
7. [`FlatRowReader`](../../core/src/main/java/dev/hardwood/internal/reader/FlatRowReader.java)
   shows direct primitive-array row access and recycling.
8. [`NestedRowReader`](../../core/src/main/java/dev/hardwood/internal/reader/NestedRowReader.java)
   consumes all-items nested batches.
9. [`ColumnReader`](../../core/src/main/java/dev/hardwood/reader/ColumnReader.java)
   documents public array/layer ownership and constructs detaching workers.
10. [`ColumnReaders`](../../core/src/main/java/dev/hardwood/reader/ColumnReaders.java)
    coordinates multiple column cursors.
11. [`PARSING_PIPELINE_V2`](../../_designs/PARSING_PIPELINE_V2.md) explains
    the intended thread-role and back-pressure design. Treat current source
    and tests as authoritative for exact ordering and data structures.

Tests:

- [`ColumnWorkerTest`](../../core/src/test/java/dev/hardwood/internal/reader/ColumnWorkerTest.java)
  directly wires real pipeline components and checks rows, batches, validity,
  nested indexes, limits, and errors.
- [`ColumnReaderBatchArrayIdentityTest`](../../core/src/test/java/dev/hardwood/ColumnReaderBatchArrayIdentityTest.java)
  pins non-aliasing for values, validity, binary buffers, and nested layers.
- [`NestedDictBatchBoundaryTest`](../../core/src/test/java/dev/hardwood/NestedDictBatchBoundaryTest.java)
  is a nested batch-boundary regression.
- [`WideListBatchBoundTest`](../../core/src/test/java/dev/hardwood/WideListBatchBoundTest.java)
  checks fan-out-aware bounded batches.
- [`RowReaderCloseIdempotencyTest`](../../core/src/test/java/dev/hardwood/RowReaderCloseIdempotencyTest.java)
  pins quiescent, idempotent child close and parent input ownership.

## Guided lab: make concurrency observable

Run from the repository root.

### 1. Simulate the reorder buffer

Set `N = 3` on paper. Submit sequence numbers 0–5, but use completion order:

```text
2, 1, 0, 4, 3, 5
```

After every completion or drain, record:

```text
nextSeq | consumePosition | slot 0 | slot 1 | slot 2 | retriever parked?
```

Do not let sequence 3 reuse slot 0 until sequence 0 has been drained. Then
compare your rule with the throttle loop in `ColumnWorker`.

### 2. Run direct flat and nested pipeline tests

```shell
timeout 180s ./mvnw -pl core -Dtest=ColumnWorkerTest test
```

Focus on:

- `flatPipelineTracksNulls`: why exact validity length is load-bearing;
- `flatPipelineRespectsMaxRows`: whether the last batch is partial;
- the nested delivery test: where derived index structures are computed;
- error propagation: how the consumer discovers a background failure.

### 3. Trace one flat page across a batch boundary

Choose `batchCapacity = 4`, an `IntPage` of six positions, and no mask.
Walk `copyPageRange`:

```text
batch 0 <- positions 0..3 -> publish
batch 1 <- positions 4..5 -> partial at EOF
```

Now add a mask `[1,3), [4,6)`. Confirm that four kept rows fill exactly one
batch even though they come from two intervals.

### 4. Inspect nested record boundaries

Use `NestedColumnWorker.assembleRegularPage` with:

```text
rep = [0,1,0,1,1,0]
```

Mark each `recordIndex`, then apply mask `[1,2)`. Identify every leaf position
copied. Check that a page boundary alone never increments a record or forces a
batch flush.

Run focused regressions:

```shell
timeout 180s ./mvnw -pl core -Dtest=NestedDictBatchBoundaryTest,WideListBatchBoundTest test
```

### 5. Prove public array ownership

Read the fixture description at the top of
`ColumnReaderBatchArrayIdentityTest`. Before running, predict which objects
must differ between batch 0 and batch 1.

```shell
timeout 180s ./mvnw -pl core -Dtest=ColumnReaderBatchArrayIdentityTest test
```

Relate each identity assertion to detaching mode. Then explain why changing
`ColumnReader` to recycling mode without copying would be an API break even if
all immediate value assertions still passed.

### 6. Verify close ownership

```shell
timeout 180s ./mvnw -pl core \
  -Dtest=RowReaderCloseIdempotencyTest,IteratorTrackingTest test
```

Trace the shutdown order:

```text
child close -> worker done -> unpark -> joins -> decode futures settle
parent close -> InputFile close
```

Then compare iterator ownership in `IteratorTrackingTest`: an exclusively
owned single-column iterator is unregistered on child close, while row-reader
and multi-column shared iterators stay tracked until parent close.

Identify the failure mode if parent close released a memory mapping before
decode futures settled.

## Common misconceptions

**“Decode tasks publish batches directly.”**  
They only publish `DecodedPage` into reorder slots. One drain thread owns
assembly state and batch publication.

**“Atomic slots automatically preserve order.”**  
The drain's `consumePosition` rule preserves order. The array only provides
bounded storage and visibility.

**“Virtual threads make CPU decoding unlimited and cheap.”**  
CPU work runs on a bounded platform-thread pool. Virtual threads coordinate
blocking stages.

**“A Parquet page boundary is a safe cross-column batch boundary.”**  
Columns have independent page boundaries. Batches align on top-level record
counts, not page ends.

**“Detaching mode has no back-pressure.”**  
It allocates fresh holders, but publication is still bounded by a two-entry
ready queue.

**“`ColumnReader` is thread-safe because returned arrays may cross threads.”**  
The arrays may be handed off after publication. The cursor's `nextBatch()` and
stateful access remain single-threaded.

## Recap and debugging checklist

- [ ] Did retriever assign one monotonic sequence number per `PageInfo`?
- [ ] Is the slot reused only after `consumePosition` advances?
- [ ] Do decode tasks publish `DecodedPage(page, mask)` and unpark drain?
- [ ] Does drain consume only the next sequence slot?
- [ ] Are file/filter boundary flags visible through the slot publication?
- [ ] Does flat validity use set-bit-present and exact active-word length?
- [ ] Does nested assembly count records from repetition level 0 rather than
      leaf positions?
- [ ] Are all projected columns cutting batches at identical row boundaries?
- [ ] Is the exchange mode consistent with the public ownership contract?
- [ ] Can a stalled consumer fill only bounded queues/pages before retrieval
      parks?
- [ ] Does an error reach `BatchExchange.checkError()` with file context?
- [ ] Does close quiesce coordinators and decode tasks before parent resources?

## Quiz

1. Why are there two long-lived virtual threads per column but no virtual
   thread per page decode?
2. With `N = 8`, sequence 9 maps to which slot, and what condition proves that
   overwriting that slot is safe?
3. Decode sequence 4 completes before sequence 3. What does drain do, and what
   wakes it when sequence 3 completes?
4. Why can `FlatColumnWorker` use `System.arraycopy` while
   `NestedColumnWorker` must inspect repetition levels in its regular path?
5. Explain why flushing every nested column at each page boundary can return
   correct per-column values yet break a multi-column reader.
6. Trace the full back-pressure chain from a full `readyQueue` to stopped I/O.
7. Which exchange mode is used for `ColumnReader`, what public promise requires
   it, and what remains bounded?
8. Why must worker close wait for in-flight decode futures after retriever and
   drain have stopped?

[Answer key](../answers/14-assembly-and-concurrency.md)
