# Answers — Chapter 12: Planning I/O and Finding Pages

[Return to the lesson](../chapters/12-io-planning.md)

## 1. Known versus discovered page boundaries

An indexed plan requires an `OffsetIndex`. Its `PageLocation`s provide each
data page's absolute offset, compressed total size, and first row index, so
planning can select page ranges before fetching page bodies.

A sequential plan has only the column-chunk offset/length and page metadata
embedded in the chunk. It must parse each variable-length `PageHeader`, measure
that header, and add `compressedPageSize` to discover the next boundary. It
also derives record progress while scanning when row masks are active.

## 2. Why `PageInfo`, not decoded `Page`?

`FetchPlan` and `PageSource` belong to I/O discovery. `PageInfo` carries the
complete page bytes plus column metadata/schema, dictionary, and row mask.
Keeping it encoded allows `ColumnWorker` to submit CPU-heavy decompression and
decoding to the shared bounded executor.

Returning decoded pages here would collapse I/O planning into CPU execution,
prevent the intended overlap, and put decode on the retriever coordination
thread.

## 3. First `ChunkHandle.slice` call

`slice` calls `ensureFetched()`. Because `data` is null, the handle enters
`fetchData`, synchronizes, checks again, and calls
`InputFile.readRange(fileOffset, length)` under the composed `FetchReason`.
It stores the returned buffer in volatile `data`.

For a standalone handle, `ensureFetched` then starts an asynchronous
`nextChunk.fetchData()` if a next handle exists. Finally, `slice` computes the
relative offset with `Math.toIntExact` and returns the requested buffer slice.
Future accesses use cached `data`.

## 4. Distant indexed pages

Pages 0 and 8 become separate `PageGroup`s because the gap is too large to
bridge. Each group gets a `ChunkHandle`, and the first handle links to the
second for one-ahead prefetch.

The plan is not cross-column-coalesce-safe because its first handle is not the
whole byte region this column will read. Expanding a shared first region
across the omitted gap could fetch dropped bytes, while the existing second
handle later fetches its range again. The conservative gate avoids over-fetch
and double-fetch.

## 5. `valuesRead` and `recordsRead`

For flat columns, one level position is one top-level record, so the counters
move together. For nested columns, one record can contain several leaf/level
positions. `valuesRead` validates progress against
`ColumnMetaData.numValues`; `recordsRead` places the page in row-group row
coordinates and computes `PageRowMask`.

Using `valuesRead` as a nested row cursor would shift masks after any
multi-element list and misalign projected columns.

## 6. Proving sequential early exit

Returned rows cannot distinguish early exit from scanning every page and
discarding trailing pages. `SequentialFetchPlanEarlyExitTest` records
`RowGroupScannedEvent.pageCount` for an all-rows plan and a bounded leading
range. The bounded event must report dramatically fewer scanned pages.

That observation pins header-scan work itself, which is the optimization's
actual invariant.

## 7. Dictionary-area fallback

If `dictionary_page_offset` is missing or nonpositive, but the first data-page
location from `OffsetIndex` is later than `ColumnMetaData.dataPageOffset`,
Hardwood treats the metadata data-page offset as the start of the preceding
dictionary area:

```text
dictAreaStart = metaData.dataPageOffset()
dictAreaEnd   = firstDataPageOffset
```

The region is then parsed and accepted only if its header is a dictionary
page. If the offsets are equal, there is no preceding region to parse.

## 8. Delayed cache eviction

Each `PageSource` may still need the shared plan, index buffers, dictionary, or
fetched `ChunkHandle` data for the work item. Evicting when the fastest column
advances would remove shared state while a slower sibling is still consuming
it. `RowGroupIterator` therefore starts a reference count at the projected
column count and evicts only at zero.

After eviction, already emitted `ByteBuffer` slices and decode tasks retain
references to their parent byte storage. Eviction removes the cache's strong
reference; it does not invalidate in-flight slices.
