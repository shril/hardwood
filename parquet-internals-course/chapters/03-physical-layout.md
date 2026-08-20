# Chapter 3 — The Physical Parquet Hierarchy

Chapter 1 explained why leaf columns are separable. Chapter 2 supplied the
language of offsets, lengths, and byte ranges. This chapter combines them into
Parquet's physical hierarchy:

```text
file -> row groups -> column chunks -> pages
```

Knowing the owner of an offset is one of the fastest ways to localize a reader
bug.

## Learning objectives

After this chapter, you should be able to:

1. distinguish files, row groups, column chunks, and pages;
2. map a table partitioned by rows and leaves onto column chunks;
3. calculate projected chunk ranges from metadata;
4. explain why a page is not a row and why pages across columns need not align;
5. place optional page indexes and bloom filters outside both page bodies and
   the serialized footer;
6. trace Hardwood metadata records from a row group to a chunk start; and
7. identify the hierarchy level responsible for a suspicious size or offset.

## Prerequisite recap

From Chapter 1:

- primitive schema leaves own stored value streams;
- projection chooses leaf columns, while predicates choose rows;
- a runtime `long[]` is not an on-disk format unit.

From Chapter 2:

- offsets are zero-based;
- `[start, end)` is a half-open byte range;
- a decoder needs bytes, a representation contract, and a cursor;
- the first and last four bytes of the small fixture spell `PAR1`.

We will use offsets and sizes in decimal unless a byte pattern itself matters.

## Central mental model: tile the table twice

Parquet partitions a table along both dimensions:

1. **horizontally** into row groups;
2. **vertically** into one column chunk per primitive leaf in each row group.

Then it divides each chunk into pages:

```text
logical table

                 leaf A       leaf B       leaf C
rows 0..999      chunk A0     chunk B0     chunk C0    <- row group 0
rows 1000..1999  chunk A1     chunk B1     chunk C1    <- row group 1
rows 2000..2999  chunk A2     chunk B2     chunk C2    <- row group 2
                    |
                    +-- page, page, page, ...
```

The **row group** owns a horizontal record interval. The **column chunk** owns
one leaf's representation over that interval. The **page** is a framed piece of
that chunk that can be encoded and compressed independently.

Keep two coordinate systems separate:

```text
logical coordinate: (record, leaf path)
physical coordinate: (file offset, byte length)
```

Footer metadata maps between them.

## The file envelope

For a conventional single-file Parquet file:

```text
lower offsets

+-------------------------------+
| PAR1 (4-byte opening magic)    |
+-------------------------------+
| row group 0, leaf 0 chunk     |
| row group 0, leaf 1 chunk     |
| ...                           |
| row group 1, leaf 0 chunk     |
| ...                           |
+-------------------------------+
| optional auxiliary structures |
| (indexes, bloom filters, etc.) |
+-------------------------------+
| FileMetaData, Thrift Compact   |
+-------------------------------+
| metadata length, 4-byte LE     |
+-------------------------------+
| PAR1 (4-byte closing magic)    |
+-------------------------------+

higher offsets
```

> **Parquet format rule:** A file begins and ends with the four bytes `PAR1`.
> Immediately before the closing magic is the four-byte little-endian length
> of the serialized file metadata; the metadata immediately precedes that
> length.

> **Parquet format rule:** File metadata is written after the column data so a
> writer can emit data in one forward pass. It records the locations needed by
> a reader.

The diagram shows the usual chunk order from the specification. The metadata
model can also refer to chunks in other files using the legacy `file_path`
field. Do not conclude that every byte belonging to a logical row group is
necessarily in the same physical file.

Also note what the footer does **not** contain. Column indexes, offset indexes,
and bloom filters are separately serialized byte regions when present. Footer
fields carry their offsets and lengths. Calling those complete structures
“inside the footer” confuses the locator with the located bytes.

## Row groups: horizontal partitions

A row group describes a contiguous logical sequence of records. If a
3,000-row file has three 1,000-row groups:

```text
row group 0 -> records 0..999
row group 1 -> records 1000..1999
row group 2 -> records 2000..2999
```

Record numbers here are conceptual; the format stores each row group's count,
not a global row number beside every record.

A row group is useful because it is:

- a coarse unit of work assignment;
- a unit that statistics can sometimes eliminate;
- a boundary at which every leaf begins a new chunk; and
- a way to bound in-memory and writer buffering.

Its metadata includes a list of column chunks, an uncompressed byte size, and
a row count. Chunks appear in schema order.

> **Parquet format rule:** Every row group contains one column chunk for every
> primitive leaf column in the schema, in the same order as the schema leaves.

For `R` row groups and `L` leaves, there are:

```text
R × L column chunks
```

This does not mean there are `R × L` Java decoder instances or network
requests. Those are reader-planning decisions.

## Column chunks: one leaf in one row group

A column chunk contains page headers and page bodies for one leaf over one row
group. Its metadata identifies at least:

- physical type;
- schema path;
- value encodings used;
- compression codec;
- value count;
- total compressed and uncompressed page sizes;
- first data-page offset; and
- optional dictionary-page offset and statistics.

The total compressed size includes page headers and compressed page data in the
chunk. It gives the extent a reader can use to bound scanning.

A chunk may begin with a dictionary page. If it does, the first data page is
later:

```text
dictionary_page_offset -> [dictionary page]
data_page_offset       -> [data page 0][data page 1]...
```

Without a dictionary:

```text
data_page_offset       -> [data page 0][data page 1]...
```

This is why the start is:

```text
dictionary_page_offset if present, otherwise data_page_offset
```

It is not always `data_page_offset`.

`num_values` counts encoded positions, including null positions. For a flat
non-repeated leaf it normally equals the row-group row count. For repeated
nested data, a row may contribute zero, one, or many positions, so value count
and row count need not match.

## Pages: framed pieces of a chunk

A page consists of a Thrift-encoded page header followed immediately by the
page body described by that header. A column chunk can contain:

- at most one dictionary page, before data pages;
- one or more data pages; and
- legacy index pages, though they are not used by modern writers.

Data pages carry repetition levels when needed, definition levels when needed,
and encoded non-null values. Compression is applied at page granularity.
Chapter 6 develops the precise V1 and V2 boundaries.

> **Parquet format rule:** Pages in a column chunk are written back-to-back.
> Page headers identify their type and compressed and uncompressed sizes.

> **Parquet format rule:** A dictionary page, when used for a chunk, must
> precede the data pages that refer to it.

Pages are not shared across leaf columns. They also are not required to cover
the same row intervals in different chunks:

```text
leaf A: [page rows 0..399] [400..799] [800..999]
leaf B: [page rows 0..249] [250..699] [700..999]
```

For flat leaves, the reader can count positions. For nested leaves, repetition
levels identify record boundaries. An optional offset index can state each
data page's starting row index, but a valid file need not contain one.

## Encoding and compression are below the hierarchy

Use four distinct questions:

| Question | Example answer |
|---|---|
| Which structural unit? | column chunk |
| Which value encoding? | PLAIN or RLE_DICTIONARY |
| Which compression codec? | UNCOMPRESSED, SNAPPY, GZIP |
| Which logical interpretation? | timestamp or string |

A page can contain PLAIN-encoded values compressed with Snappy. PLAIN is not
“uncompressed,” and Snappy does not determine whether values are dictionary
indices.

## Worked physical example

Suppose a six-row, three-leaf table has two row groups:

```text
leaves in schema order: id, city, total
row group 0: rows 0..2
row group 1: rows 3..5
```

Assume the footer reports these compressed chunk ranges:

| row group | leaf | start | compressed size | range |
|---:|---|---:|---:|---|
| 0 | `id` | 4 | 80 | `[4, 84)` |
| 0 | `city` | 84 | 70 | `[84, 154)` |
| 0 | `total` | 154 | 90 | `[154, 244)` |
| 1 | `id` | 244 | 80 | `[244, 324)` |
| 1 | `city` | 324 | 70 | `[324, 394)` |
| 1 | `total` | 394 | 90 | `[394, 484)` |

The rest of the synthetic file is:

```text
[484, 520) optional indexes
[520, 680) serialized FileMetaData
[680, 684) metadata length = 160, little-endian
[684, 688) PAR1
```

Check the arithmetic for row group 0:

```text
id end    = 4 + 80   = 84
city end  = 84 + 70  = 154
total end = 154 + 90 = 244
```

A projection of only `city` needs these data regions:

```text
[84, 154)   -> 70 bytes
[324, 394)  -> 70 bytes
total chunk bytes = 140
```

It also needs the file envelope and footer reads used to discover those
locations. “Projection reads 140 bytes” would therefore be incomplete;
“projection reads 140 bytes of column-chunk data” is precise.

For:

```sql
SELECT total FROM orders WHERE city = 'Oslo';
```

a plan may need:

```text
city:  [84,154) and [324,394)   predicate input
total: [154,244) and [394,484)  projected payload
```

If row-group `city` statistics prove row group 1 cannot contain `Oslo`, the
reader may skip both row group 1 chunks. That optimization relies on metadata,
not on changing the hierarchy.

Suppose the `city` chunk in row group 0 has:

```text
dictionary_page_offset = 84
data_page_offset       = 109
total_compressed_size  = 70
```

Its chunk still begins at 84 and ends at:

```text
84 + 70 = 154
```

Starting at 109 would omit the dictionary needed by dictionary-encoded data
pages.

## Format rules and implementation choices

> **Parquet format rule:** The hierarchy is file, row group, column chunk,
> page. A chunk belongs to one leaf and one row group.

> **Parquet format rule:** Page-index and bloom-filter byte regions are
> optional. Their locators are metadata fields; absence is valid.

> **Parquet format rule:** A column chunk may name another file through legacy
> split-file metadata.

> **Hardwood implementation choice:** `ColumnChunk.chunkStartOffset()` prefers
> a positive dictionary offset and otherwise returns the data-page offset.

> **Hardwood implementation choice:** Hardwood can inspect metadata for a
> split-file chunk but refuses to read its data with `requireSameFile()`.

> **Hardwood implementation choice:** `RowGroupIterator` turns selected
> `(file, row group)` pairs into work, and `PageSource` hides whether pages came
> from an offset index or sequential discovery.

> **Hardwood implementation choice:** Hardwood's default local input can map a
> large file, but each compressed chunk is constrained by Java's addressable
> in-memory region sizes. That is not a Parquet file-size rule.

## Current Hardwood code tour

1. [`FileMetaData`](../../core/src/main/java/dev/hardwood/metadata/FileMetaData.java)
   owns flattened schema elements, total row count, and row groups.
2. [`RowGroup`](../../core/src/main/java/dev/hardwood/metadata/RowGroup.java)
   owns its chunk list, uncompressed byte size, and row count.
3. [`ColumnChunk`](../../core/src/main/java/dev/hardwood/metadata/ColumnChunk.java)
   owns `ColumnMetaData` and optional auxiliary-index locations. Read
   `chunkStartOffset()` and `requireSameFile()`.
4. [`ColumnMetaData`](../../core/src/main/java/dev/hardwood/metadata/ColumnMetaData.java)
   carries type, encoding list, path, codec, counts, sizes, page offsets, and
   optional statistics.
5. [`RowGroupReader`](../../core/src/main/java/dev/hardwood/internal/thrift/RowGroupReader.java)
   decodes row-group fields and notes that chunks arrive in schema order.
6. [`ColumnMetaDataReader`](../../core/src/main/java/dev/hardwood/internal/thrift/ColumnMetaDataReader.java)
   maps numbered metadata fields onto the runtime record.
7. [`RowGroupIterator`](../../core/src/main/java/dev/hardwood/internal/reader/RowGroupIterator.java)
   builds work items and fetch plans for projected columns.
8. [`PageSource`](../../core/src/main/java/dev/hardwood/internal/reader/PageSource.java)
   exposes a per-column stream of `PageInfo` while hiding indexed versus
   sequential page location.

The useful trace is:

```text
FileMetaData
  -> RowGroup
    -> ColumnChunk
      -> ColumnMetaData offsets and sizes
        -> FetchPlan
          -> PageSource
```

## Guided lab: follow a three-row-group fixture

This lab uses
`core/src/test/resources/filter_pushdown_int.parquet`, whose checked-in test
documents three row groups containing ids 1–100, 101–200, and 201–300.

### 1. Verify the documented partition

Open
[`RowGroupFilterTest`](../../core/src/test/java/dev/hardwood/RowGroupFilterTest.java)
and inspect `readMidpoints()` and `midpoint(RowGroup)`.

Record:

- the expected number of row groups;
- the first chunk's start expression;
- how the test approximates a row group's physical midpoint;
- why that midpoint is an application policy rather than a stored Parquet
  record identifier.

### 2. Trace the start calculation

```shell
rg -n "chunkStartOffset|dictionaryPageOffset|dataPageOffset" \
  core/src/main/java/dev/hardwood/metadata/ColumnChunk.java
```

Evaluate the method for:

| dictionary offset | data offset | result |
|---:|---:|---:|
| 1,200 | 1,240 | ? |
| absent | 1,240 | ? |
| 0 | 1,240 | ? |

The results are 1,200, 1,240, and 1,240. Hardwood treats a non-positive
dictionary offset as unusable.

### 3. Relate row groups to values

Run one focused test:

```shell
timeout 180s ./mvnw -pl core \
  -Dtest=RowGroupFilterTest#columnReaderByteRangeKeepsOnlyRowGroupsByMidpointRule \
  test
```

Inspect the assertions around that method. Explain why selecting a physical
range at row-group granularity returns a contiguous logical id interval even
though values are stored by column.

### 4. Trace hierarchy ownership

Use these focused searches:

```shell
rg -n "record FileMetaData|record RowGroup|record ColumnChunk|record ColumnMetaData" \
  core/src/main/java/dev/hardwood/metadata
rg -n "getColumnPlan|PageSource" \
  core/src/main/java/dev/hardwood/internal/reader
```

Create a four-line note assigning each item to its owner:

```text
total file rows       -> FileMetaData
rows in one partition -> RowGroup
offset-index location -> ColumnChunk
first data-page offset -> ColumnMetaData
```

### 5. Check the file envelope

```shell
FILE=core/src/test/resources/filter_pushdown_int.parquet
SIZE=$(stat -c %s "$FILE")
od -An -tx1 -N4 "$FILE"
od -An -tx1 -j $((SIZE-4)) -N4 "$FILE"
```

Both boundaries should be `50 41 52 31`, the bytes for `PAR1`. Do not try to
find row-group boundaries by scanning for this marker; their locations come
from metadata.

## Common misconceptions

**“A row group stores rows one after another.”**  
It owns a range of rows, but stores one chunk per leaf.

**“A column chunk is the whole file's column.”**  
It is one leaf in one row group. The same leaf has another chunk in every row
group.

**“A page is a horizontal batch of complete rows.”**  
No. A page belongs to one leaf. Pages in sibling leaves may have different
boundaries.

**“The page index is inside the footer.”**  
The footer stores page-index offsets and lengths. The index bytes are separate
regions.

**“Compression codec and encoding identify the same operation.”**  
Encoding transforms typed values; compression transforms the resulting bytes.

**“The first data-page offset always starts the chunk.”**  
Not when a dictionary page precedes it.

**“Row-group byte size is enough to locate every page.”**  
No. Chunk and page metadata provide the relevant boundaries; without an offset
index, a reader scans page headers within the bounded chunk.

## Recap and debug checklist

- Which file and row group own the bad record?
- Which primitive leaf and column chunk own the bad value?
- Did chunk order match schema leaf order?
- Did the chunk start at a dictionary page or first data page?
- Is `totalCompressedSize` being used as a bounded extent?
- Is the suspicious number a row count, value count, page count, or byte size?
- Is an optional index absent, or merely stored outside the footer?
- Am I assuming sibling pages align?
- Does metadata name another physical file?

## Quiz

1. Define a row group, column chunk, and page in one sentence each.
2. A schema has seven primitive leaves and a file has four row groups. How many
   column chunks must the row-group metadata describe? Explain.
3. In the worked example, which chunk-data ranges and how many chunk bytes are
   needed for projection `id, total` across both row groups?
4. A chunk has `dictionaryPageOffset = 900`,
   `dataPageOffset = 940`, and `totalCompressedSize = 180`. What half-open
   range bounds the chunk? What would be lost by starting at 940?
5. Leaf A has two data pages while leaf B has five in the same row group. Is
   the file necessarily malformed? How can a reader still reconstruct records?
6. Trace `ColumnChunk.chunkStartOffset()` for (a) dictionary offset 700 and data
   offset 750, (b) absent dictionary offset and data offset 750, and (c)
   dictionary offset 0 and data offset 750.
7. Where are an offset index's bytes stored relative to the `FileMetaData`
   object, and what does the footer retain?
8. Classify each as a **format rule** or a **Hardwood choice**:
   (a) one chunk per leaf per row group; (b) reject reading a non-empty
   `filePath`; (c) dictionary page precedes data pages; (d) expose pages through
   a `PageSource` iterator.

Compare your answers with
[`../answers/03-physical-layout.md`](../answers/03-physical-layout.md).
