# Answers — Chapter 3: The Physical Parquet Hierarchy

Return to
[`../chapters/03-physical-layout.md`](../chapters/03-physical-layout.md).

## 1. The three nested units

- A **row group** is a horizontal partition that owns a contiguous logical
  range of records and one chunk for every schema leaf.
- A **column chunk** is the pages for one primitive leaf within one row group.
- A **page** is an independently headed and usually independently compressed
  piece inside one column chunk.

The ownership chain is row group → column chunk → page.

## 2. Chunk count

Every row group has one chunk for every primitive leaf:

```text
4 row groups × 7 leaves = 28 column chunks
```

The count is about logical units represented by metadata, not necessarily I/O
requests. A reader may coalesce adjacent ranges.

## 3. Projecting `id, total`

From row group 0:

```text
id:    [4, 84)    -> 80 bytes
total: [154, 244) -> 90 bytes
```

From row group 1:

```text
id:    [244, 324) -> 80 bytes
total: [394, 484) -> 90 bytes
```

Total chunk bytes:

```text
80 + 90 + 80 + 90 = 340 bytes
```

The answer excludes opening magic and metadata reads. The four chunk ranges
need not become four physical requests if an implementation chooses to
coalesce ranges.

## 4. Dictionary at the chunk start

The dictionary offset is present and earlier, so chunk start is 900. The end
is:

```text
900 + 180 = 1,080
```

The bounded range is:

```text
[900, 1,080)
```

Starting at 940 would omit the dictionary page. Data pages containing
dictionary indices would then lack the values those indices identify.

## 5. Different page counts across leaves

The file is not necessarily malformed. Pages belong to one leaf and writers
may split each leaf according to encoded size or other policies, so sibling
chunks need not have aligned page counts or boundaries.

A reader preserves record coordinates using value counts and, where nesting or
nullability requires them, definition and repetition levels. An offset index
can additionally provide a page's first row index, but valid files may omit it.

## 6. `chunkStartOffset()` traces

1. Dictionary 700, data 750 → **700**. A positive dictionary offset starts the
   chunk.
2. Dictionary absent, data 750 → **750**.
3. Dictionary 0, data 750 → **750**. Hardwood's method accepts the dictionary
   offset only when it is non-null and positive.

The positive-value condition is Hardwood behavior around metadata; the format
relationship is that an actual dictionary page precedes referring data pages.

## 7. Offset-index location

The offset-index bytes are a separately serialized region in the file, usually
placed near the end among auxiliary structures. They are not nested inside the
serialized `FileMetaData` bytes.

The footer retains each index region's offset and length in the relevant
`ColumnChunk` metadata. A reader uses that locator to perform a second range
read.

## 8. Format rule or Hardwood choice

1. **Format rule:** every row group has one column chunk per primitive schema
   leaf.
2. **Hardwood choice:** Hardwood can parse a non-empty `filePath` but rejects a
   data read that would need to follow it. The Parquet field is used by summary
   `_metadata` files; it does not define arbitrary externalized chunks as the
   normal layout of a data file.
3. **Format rule:** a dictionary page, when present, precedes data pages that
   use it.
4. **Hardwood choice:** presenting pages to workers through a `PageSource`
   iterator is reader architecture, not an on-disk requirement.
