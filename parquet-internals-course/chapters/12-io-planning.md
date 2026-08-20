# Chapter 12 — Planning I/O and Finding Pages

**Study time:** 45–75 minutes

Opening supplied trustworthy metadata. The next problem is selective movement:
given a projection, filter, row limit, and sequence of files, which page bytes
should Hardwood fetch, when should it fetch them, and how does one column see a
uniform page stream despite two different discovery strategies?

## Objectives

By the end of this lesson, you should be able to:

1. explain the division of responsibility among `RowGroupIterator`,
   `FetchPlan`, `ChunkHandle`, `DictionaryParser`, and `PageSource`;
2. contrast an indexed plan based on `OffsetIndex` with a sequential plan that
   discovers page boundaries from headers;
3. trace lazy I/O, within-column prefetch, next-row-group prefetch, and cache
   release;
4. explain why dictionary bytes may precede the first data page and how both
   plan types make the same `Dictionary` available to decoding;
5. identify the alignment rules that constrain filtering, row masks, and
   cross-column coalescing.

## Prerequisite recap

- A row group contains one column chunk per leaf column.
- A column chunk starts at a metadata-derived offset and contains dictionary,
  data, and possibly index page records.
- Every page starts with a variable-length Thrift header. The header states
  the compressed body size, so the next page begins after
  `headerSize + compressedPageSize`.
- An `OffsetIndex`, when present, gives a `PageLocation` for each data page:
  offset, compressed size, and first row index.
- A dictionary page is separate from dictionary-encoded data pages. The data
  pages carry indices; decoding requires the dictionary values as context.
- Projection is by leaf column, but rows emitted by different projected
  columns must stay aligned.

## Central mental model: plan globally, expose locally

The current architecture is:

```text
                          shared across projected columns
             +---------------------------------------------+
             | RowGroupIterator                            |
             | work list: (file, row group)                |
             | schema/filter/index/mask state              |
             | FetchPlan[] computed once per work item     |
             +---------------------------------------------+
                    | plan 0       | plan 1       | plan N
                    v              v              v
               PageSource     PageSource     PageSource
               (column 0)     (column 1)     (column N)
                    |              |              |
                    +------ PageInfo streams -----+
```

`RowGroupIterator` sees cross-file and cross-column facts. It can validate
schemas, filter whole row groups, combine page-index results, and decide
whether first reads can be coalesced. A `PageSource` sees only one projected
column. It advances through the same immutable work list and asks for that
column's `FetchPlan`.

The plan hides page-location strategy:

```text
FetchPlan
  |
  +-- IndexedFetchPlan
  |     locations known now from OffsetIndex
  |     page bytes fetched later through ChunkHandle
  |
  `-- SequentialFetchPlan
        locations learned later by parsing page headers
        headers and page bytes fetched through ChunkHandle

PageSource.next() -> PageInfo or null
```

The consumer of `PageSource` does not branch on index availability. That
separation is a central architectural invariant from
`PARSING_PIPELINE_V2.md`.

## Worked control/data-flow example

Assume one row group has 10,000 rows and projects `id` and `category`. A filter
has produced matching row range `[0, 1000)`.

### Column `id`: OffsetIndex available

Its page locations are:

```text
page 0: offset 1000, size 800, firstRowIndex 0
page 1: offset 1800, size 820, firstRowIndex 1024
...
page 9: offset 8400, size 700, firstRowIndex 9216
```

`RowGroupIterator.computeFetchPlans`:

1. reads the already-fetched index buffer into an `OffsetIndex`;
2. compares each page row interval with `[0, 1000)`;
3. keeps page 0 and associates a `PageRowMask` that retains its first 1,000
   records;
4. coalesces needed pages into page groups;
5. creates one lazy `ChunkHandle` for the required bytes;
6. builds `IndexedFetchPlan` without reading those bytes.

When `PageSource` first advances the plan, `IndexedFetchPlan.PageIterator`
locates and parses a preceding dictionary region if metadata and the first
data-page offset show one exists. It then slices page 0 from the chunk handle
and returns:

```text
PageInfo(
    pageData = header + compressed body slice,
    columnSchema,
    columnMetaData,
    dictionary,
    mask = [0, 1000)
)
```

### Column `category`: no OffsetIndex

`RowGroupIterator` builds a `SequentialFetchPlan` over the column chunk. On
iteration:

1. the plan creates/fetches its first chunk;
2. it parses the first page header;
3. if that header identifies a dictionary page, it reads the full dictionary
   page and uses `DictionaryParser`;
4. it advances `position` by header size plus compressed size;
5. for each data page, it derives `numValues`, determines a per-page row mask,
   and either skips, substitutes, or emits the page;
6. once `recordsRead >= matchingRows.endRow()`, it exits before parsing
   trailing headers.

For flat `category`, `numValues` is also the page's top-level record count. For
nested data, V2 repetition-level bytes can be counted without decompressing
the value region. A nested V1 page cannot provide that cheap count; the
row-group-wide mask-capability gate must disable masking for every projected
column rather than let sibling columns diverge.

### Lazy chunk fetch and prefetch

For a standalone handle:

```text
page asks handle.slice(...)
  -> ensureFetched()
       -> readRange(fileOffset, length)
       -> cache the first successful ByteBuffer
       -> asynchronously fetch nextChunk, one ahead
  -> slice relative region
```

Successful data is reused. A failed asynchronous prefetch is not cached as a
successful fetch: when that handle is later demanded, `ensureFetched()` retries
the `readRange` instead of making a transient prefetch failure permanent.

For a region-backed handle created by cross-column coalescing,
`ensureFetched()` slices a shared region. The shared region performs the
underlying read once.

Planning is metadata work; calling `pages()` creates an iterator; advancing
the iterator is where demand can become I/O. `IndexedFetchPlan.prefetch()` is
an explicit exception used for next-row-group overlap. A sequential plan's
default `prefetch()` is a no-op, though advancing its current handle chains
one-ahead fetches.

## Indexed and sequential plans side by side

| Question | `IndexedFetchPlan` | `SequentialFetchPlan` |
|---|---|---|
| How are page boundaries known? | `OffsetIndex.PageLocation` | Parse each page header |
| Can pages be selected before scanning data headers? | Yes | No |
| When are data bytes fetched? | First page/chunk access or explicit first-chunk prefetch | As header scanning advances |
| How is a dictionary found? | Region before first data-page location, using metadata fallback | First page is probed while scanning |
| How are distant kept pages read? | Coalesced into bounded page groups | Fixed/dynamically sized sequential chunks |
| How does `maxRows` reduce work? | Truncates needed page locations and masks the boundary page | Shrinks chunk estimate and stops after counters reach the budget |
| Can trailing pages be avoided? | They never enter `neededPages` | Early exit when row/value cursor reaches its bound |
| What does `PageSource` see? | `Iterator<PageInfo>` | `Iterator<PageInfo>` |

Neither strategy changes the page's on-disk encoding. They are I/O-planning
strategies for reaching the same bytes.

## Parquet format rules and Hardwood choices

### Format rules

- Page headers and compressed body sizes delimit pages in a column chunk.
- A dictionary page, if used, precedes dictionary-encoded data that depends on
  it.
- `ColumnMetaData` supplies column-chunk sizes and data/dictionary offsets.
- `OffsetIndex` is optional. When present, its page locations describe data
  pages and row starts.
- Data Page V2 leaves repetition and definition level regions uncompressed,
  which can make top-level record counting possible without value
  decompression.

### Hardwood choices

- `RowGroupIterator` builds a shared ordered work list and caches one
  `FetchPlan[]` per work item.
- Missing `OffsetIndex` selects `SequentialFetchPlan`; it does not reject the
  file.
- Indexed needed pages are coalesced when byte gaps are at most 1 MiB, with a
  bounded maximum span.
- Sequential chunks normally have a 128 MiB ceiling. With a row limit, size is
  estimated from compressed bytes per value, a safety factor, and a 1 MiB
  floor; sufficiently small chunks are read whole.
- `ChunkHandle` thread-safely caches its first successful fetch and starts
  one-ahead asynchronous prefetch for a following standalone chunk. A failed
  prefetch may be retried on demand.
- First reads from adjacent columns may share a `SharedRegion`, but only when
  each plan declares that rewrite coalesce-safe.
- The page-mask gate is row-group-wide. If any projected column cannot honor
  a nontrivial row selection safely, all plans fall back to `RowRanges.ALL`
  and exact filtering happens later.
- `PageSource` releases a work-item reference when that column advances. Once
  all projected columns release it, index metadata, plans, and fetched chunk
  references are evicted.

## Current code tour

1. [`RowGroupIterator`](../../core/src/main/java/dev/hardwood/internal/reader/RowGroupIterator.java)
   is the shared planner. Start at `initialize`, then trace `getColumnPlan`,
   `computeFetchPlans`, `computeNeededPages`, coalescing, prefetch, and
   `releaseWorkItem`.
2. [`FetchPlan`](../../core/src/main/java/dev/hardwood/internal/reader/FetchPlan.java)
   is the small strategy boundary: `isEmpty`, `pages`, and optional
   `prefetch`.
3. [`IndexedFetchPlan`](../../core/src/main/java/dev/hardwood/internal/reader/IndexedFetchPlan.java)
   turns known `PageLocation`s into lazy page slices. Study dictionary-region
   discovery and the one-page-group coalescing gate.
4. [`SequentialFetchPlan`](../../core/src/main/java/dev/hardwood/internal/reader/SequentialFetchPlan.java)
   scans headers, handles chunk crossings, advances value/record cursors,
   supports masks and inline-stat placeholders, and validates final counts.
5. [`ChunkHandle`](../../core/src/main/java/dev/hardwood/internal/reader/ChunkHandle.java)
   is the lazy range-fetch unit. Its double-checked synchronization,
   `FetchReason` propagation, and region-backed branch are all load-bearing.
6. [`SharedRegion`](../../core/src/main/java/dev/hardwood/internal/reader/SharedRegion.java)
   implements a cross-column ranged read and per-column slicing.
7. [`DictionaryParser`](../../core/src/main/java/dev/hardwood/internal/reader/DictionaryParser.java)
   parses a dictionary page header, checks CRC, decompresses, and creates a
   typed `Dictionary`.
8. [`PageInfo`](../../core/src/main/java/dev/hardwood/internal/reader/PageInfo.java)
   is the page hand-off: bytes, schema, metadata, dictionary, and mask, or a
   null placeholder.
9. [`PageSource`](../../core/src/main/java/dev/hardwood/internal/reader/PageSource.java)
   flattens work-item plans into one per-column sequence.
10. [`PARSING_PIPELINE_V2`](../../_designs/PARSING_PIPELINE_V2.md) explains
    the intended component boundaries and I/O/decode overlap. Use current
    source and tests for exact chunk-sizing and prefetch behavior.

Tests that expose planning rather than only final values:

- [`PageRangeIoTest`](../../core/src/test/java/dev/hardwood/internal/reader/PageRangeIoTest.java)
  compares bytes read with and without page-index filtering.
- [`SequentialFetchPlanEarlyExitTest`](../../core/src/test/java/dev/hardwood/internal/reader/SequentialFetchPlanEarlyExitTest.java)
  uses a JFR page count to prove header scanning stopped early.
- [`SequentialFetchPlanChunkSizeTest`](../../core/src/test/java/dev/hardwood/internal/reader/SequentialFetchPlanChunkSizeTest.java)
  pins row-limit chunk sizing.
- [`CrossColumnCoalesceTest`](../../core/src/test/java/dev/hardwood/internal/reader/CrossColumnCoalesceTest.java)
  checks shared first reads.
- [`DictionaryTest`](../../core/src/test/java/dev/hardwood/internal/reader/DictionaryTest.java)
  tests dictionary parsing edge cases.

## Guided lab: observe planning, not just correct rows

Run from the repository root.

### 1. Trace the indexed branch

Open `RowGroupIterator.computeFetchPlans` and follow the branch where
`offsetIndex() != null`. Build a table with these columns:

```text
input metadata | derived object | may cause I/O? | lifetime
```

Include `OffsetIndex`, `NeededPage`, `PageGroup`, `ChunkHandle`,
`IndexedFetchPlan`, and `PageInfo`. Mark only actual demand/prefetch operations
as I/O.

### 2. Prove page-range I/O with a real fixture

`PageRangeIoTest` uses
`core/src/test/resources/column_index_pushdown.parquet`: 10,000 sorted IDs and
roughly ten indexed pages. The filter `id < 1000` should need only the leading
page range.

Predict which result changes and which does not:

- returned matching rows;
- bytes read by `CountingInputFile`;
- Parquet page encoding.

Then run:

```shell
timeout 180s ./mvnw -pl core -Dtest=PageRangeIoTest test
```

The important assertion is not merely correct values. It is
`filteredBytes < unfilteredBytes`.

### 3. Trace the sequential branch

Read `SequentialFetchPlan.SequentialPageIterator` in this order:

1. `hasNext`;
2. `initialize`;
3. `readPageHeader`;
4. `findNextEmittablePage`;
5. `readBytes` and `advanceChunk`;
6. final count checks.

For `core/src/test/resources/misaligned_pages_no_index.parquet`, explain why
the bounded `[0, 200)` plan should scan fewer headers than `RowRanges.ALL`.

```shell
timeout 180s ./mvnw -pl core -Dtest=SequentialFetchPlanEarlyExitTest test
```

Note why the test inspects a JFR `pageCount`: returned rows alone cannot prove
that trailing headers were not scanned.

### 4. Follow a dictionary through both strategies

Compare:

- `IndexedFetchPlan.PageIterator.parseDictionary`;
- `SequentialFetchPlan.SequentialPageIterator.initialize`;
- `DictionaryParser.parse`.

Answer these before running tests:

1. Which bytes are CRC-checked?
2. Which codec is selected?
3. Which object is attached to every emitted dictionary-encoded `PageInfo`?

```shell
timeout 180s ./mvnw -pl core -Dtest=DictionaryTest,PageRangeIoTest#testColumnReaderPageRangeIoWithDictionary test
```

### 5. Check coalescing and release

Read `CrossColumnCoalesceTest` and predict when a plan refuses shared-region
rewriting. Then run:

```shell
timeout 180s ./mvnw -pl core -Dtest=CrossColumnCoalesceTest test
```

Finally, draw two projected `PageSource`s advancing at different rates.
`PageSource A` may release work item 0 before `PageSource B`; caches are
evicted only after B also releases it.

## Common misconceptions

**“An OffsetIndex contains decoded values.”**  
It contains page locations and first-row positions, not page values.

**“Creating an indexed plan reads all selected pages.”**  
Plan construction is metadata-only. A `ChunkHandle` fetches on demand or
explicit prefetch.

**“Sequential means one `readRange` per page.”**  
Header scanning and page slicing operate over bounded chunks. Multiple pages
normally share one fetched chunk.

**“No OffsetIndex means no page-level optimization is possible.”**  
Sequential scanning can stop for row limits/ranges and can use inline page
statistics. It simply cannot jump to arbitrary page offsets without walking
preceding headers.

**“One projected column may apply a row mask even if a sibling cannot.”**  
That would misalign batches. Hardwood's capability gate applies the decision
to the row group as a whole.

**“A dictionary is metadata in the footer.”**  
Dictionary encoding is declared in metadata, but dictionary values live in a
dictionary page in the column chunk.

## Recap and debugging checklist

- [ ] Did `RowGroupIterator` include the correct `(file, row group)` work item?
- [ ] Was the projected column translated to the correct file-local ordinal?
- [ ] Were index buffers sliced for that exact row group and column?
- [ ] Did the presence or absence of `OffsetIndex` choose the expected plan?
- [ ] For indexed I/O, do `NeededPage` masks and byte ranges match page rows?
- [ ] For sequential I/O, do `position`, `valuesRead`, and `recordsRead`
      advance even for dropped pages?
- [ ] Is dictionary discovery based on the first data-page boundary or first
      scanned page, as appropriate?
- [ ] Does `ChunkHandle` reuse a successful fetch, retry failed prefetch on
      demand, preserve file attribution, and avoid unsafe coalescing?
- [ ] Can every projected column honor the same row mask?
- [ ] Does `PageSource` release each work item exactly once when advancing?
- [ ] Does the test measure I/O or page scans when the bug is wasted work?

## Quiz

1. What information makes an indexed plan possible, and what information must
   a sequential plan discover itself?
2. Why does `FetchPlan.pages()` return `PageInfo` rather than decoded `Page`
   objects?
3. Trace the first call to `ChunkHandle.slice` when neither it nor its
   `nextChunk` has been fetched.
4. An indexed plan keeps pages 0 and 8 with a gap larger than the coalescing
   threshold. How are handles organized, and why is cross-column coalescing
   conservative in this case?
5. Why does a sequential iterator maintain both `valuesRead` and
   `recordsRead` for nested pages?
6. How can `SequentialFetchPlanEarlyExitTest` distinguish “returned only
   leading rows” from “scanned everything and discarded trailing pages”?
7. When can metadata's `data_page_offset` act as the dictionary-area start
   even if `dictionary_page_offset` is absent?
8. Two `PageSource`s share a work item. One finishes it much earlier. Why
   must plan-cache eviction wait for both, and what keeps already emitted
   page slices alive after eviction?

[Answer key](../answers/12-io-planning.md)
