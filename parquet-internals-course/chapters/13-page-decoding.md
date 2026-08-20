# Chapter 13 — Decoding a Page

**Study time:** 45–75 minutes

The I/O layer yields complete page byte slices plus schema, metadata,
dictionary, and row-mask context. This chapter follows the CPU-bound boundary:
the exact sequence by which `PageDecoder` validates, decompresses, decodes
levels and values, and returns a typed array-backed `Page`.

## Objectives

By the end of this lesson, you should be able to:

1. trace `PageDecoder.decodePage` in source order without confusing V1 and V2
   body layouts;
2. explain why CRC validation precedes decompression and why V2 decompresses
   only its value region;
3. derive the primitive output array and null slots for a small page;
4. explain the `Page.size()`, pooled-level-array, and all-present invariants;
5. map an encoding/physical-type pair to the decoder family or controlled
   unsupported path.

## Prerequisite recap

Three transformations must remain separate:

```text
compression: compressed page bytes -> uncompressed encoded bytes
encoding:    encoded values        -> physical values
logical type: physical values      -> user-facing interpretation
```

Page decoding performs the first two. Logical conversion belongs downstream.

Also recall:

- The page header is Thrift Compact Protocol and is not part of the compressed
  body.
- `num_values` counts level positions, including null/empty structural
  positions. Fewer physical values may be encoded.
- Definition levels identify which positions reach the leaf's maximum
  definition level.
- Repetition levels preserve nested record/list boundaries.
- Dictionary data pages contain encoded dictionary indices; the dictionary
  page was parsed earlier.

## Central mental model: structure before values

For every normal data page:

```text
complete page ByteBuffer
        |
        v
parse PageHeader; measure header bytes
        |
        v
slice compressed body
        |
        v
validate optional CRC over stored body
        |
        +---------------------+
        |                     |
        v                     v
   DATA_PAGE (V1)        DATA_PAGE_V2
 decompress whole body   levels stay uncompressed
 split length-prefixed   split by header byte lengths
 level regions           decompress value region only,
        |                 if is_compressed
        +----------+----------+
                   v
          decode repetition levels
          decode definition levels
                   |
                   v
       choose decoder from Encoding + PhysicalType
                   |
                   v
       Page.IntPage / LongPage / ... / ByteArrayPage
```

The ordering is semantic, not cosmetic. The decoder needs levels before
values because null positions consume no encoded value. It needs the page
header before decompression because the header selects V1/V2, lengths,
encoding, and uncompressed size. It validates the CRC over stored compressed
body bytes because that is the page checksum's domain.

## Worked example: optional INT32 in a V1 page

Assume the schema has:

```text
optional int32 quantity
maxDefinitionLevel = 1
maxRepetitionLevel = 0
```

The data page header says:

```text
type = DATA_PAGE
num_values = 4
encoding = PLAIN
definition_level_encoding = RLE
compressed_page_size = 18
uncompressed_page_size = 18
codec = UNCOMPRESSED
```

After the header, the V1 body contains:

```text
+---------------------+----------------------+----------------------+
| def stream length=2 | def bytes -> 1,0,1,1 | PLAIN ints 10,20,30 |
+---------------------+----------------------+----------------------+
  4 little-endian bytes       2 bytes              12 bytes
```

There is no repetition-level region because `maxRepetitionLevel == 0`.

The exact trace is:

1. `ThriftCompactReader` parses `PageHeader`; `getBytesRead()` gives
   `headerSize`.
2. `pageBuffer.slice(headerSize, 18)` isolates the body.
3. If a CRC exists, `CrcValidator` checks those 18 stored bytes.
4. V1 asks the column codec's `Decompressor` for 18 uncompressed bytes. The
   uncompressed codec still follows the same interface.
5. `parseDataPage` sees no repetition levels. It reads the first little-endian
   4-byte definition-stream length, records that region, and advances
   `valuesOffset`.
6. `decodeDefinitionLevels` calculates bit width 1 and produces
   `[1, 0, 1, 1]`.
7. `decodeTypedValues(PLAIN, INT32, ...)` allocates `int[4]`.
8. `PlainDecoder.readInts` consumes an encoded integer only where the
   definition level equals 1.

The result is conceptually:

```text
IntPage
  values           = [10, 0, 20, 30]
  definitionLevels = [ 1, 0,  1,  1]
  repetitionLevels = null
  maxDefinitionLevel = 1
  size = 4
  fixedListK = 0
```

The zero at `values[1]` is an unused primitive-array default, not the SQL value
zero. Consumers must consult presence before reading it.

### The all-present variant

If the definition stream is one RLE run of four `1`s,
`decodeDefinitionLevels` returns `null` instead of materializing
`[1,1,1,1]`. Hardwood's convention is:

```text
definitionLevels == null  <=>  all leaf positions are present
```

This applies to required columns and to optional/nested pages proved entirely
present. Nested pages can still have non-null repetition levels.

### The V2 variation

A Data Page V2 header gives:

```text
repetition_levels_byte_length
definition_levels_byte_length
is_compressed
num_values
num_rows
encoding
```

Its body is:

```text
[uncompressed rep levels][uncompressed def levels][possibly compressed values]
```

`parseDataPageV2` slices/copies the level regions first. It decodes them, then
`readValueRegion` decompresses only the value bytes if `is_compressed` is
true. The `uncompressedPageSize` includes levels, so expected uncompressed
value length is:

```text
uncompressedPageSize - repLevelLength - defLevelLength
```

It would be wrong to decompress the entire V2 body as one codec stream.

## Encoding dispatch and output shapes

`decodeTypedValues` dispatches first on encoding, then where needed on
physical type:

| Encoding | Relevant output families |
|---|---|
| `PLAIN` | every supported physical type |
| `DELTA_BINARY_PACKED` | `INT32`, `INT64` |
| `BYTE_STREAM_SPLIT` | numeric types and `FIXED_LEN_BYTE_ARRAY` supported by the implementation |
| `RLE_DICTIONARY`, `PLAIN_DICTIONARY` | dictionary controls typed result |
| `RLE` | Boolean values in the current implementation |
| `DELTA_LENGTH_BYTE_ARRAY` | byte arrays |
| `DELTA_BYTE_ARRAY` | byte arrays |

The sealed `Page` hierarchy mirrors physical storage:

```text
BOOLEAN                 -> BooleanPage(boolean[])
INT32                   -> IntPage(int[])
INT64                   -> LongPage(long[])
FLOAT                   -> FloatPage(float[])
DOUBLE                  -> DoublePage(double[])
BYTE_ARRAY,
FIXED_LEN_BYTE_ARRAY,
INT96                   -> ByteArrayPage(byte[][])
```

The numeric and Boolean paths avoid boxing. Binary values use `byte[][]`
because each value can have separate length. Dictionary-backed
`ByteArrayPage` can additionally preserve a shared
`ByteArrayDictionary` and per-position `dictIndices`, allowing downstream
batch assembly to intern repeated dictionary values rather than clone each
byte array.

For all page records:

- `size()` is the logical number of positions to process.
- Value arrays are indexed by logical position and therefore have
  `num_values` slots, including unused null slots.
- A non-null definition/repetition level array may be physically longer than
  `size()`. Decode slots own reusable `LevelScratch` arrays that grow but do
  not shrink.
- Only indices `[0, size())` are valid. Array length is not a row/value count.
- `fixedListK > 0` marks a detected fixed-width list page whose shape makes
  explicit levels unnecessary on the optimized path.

## Dictionary and placeholder paths

The normal dictionary path reads:

1. one unsigned byte of dictionary-index bit width;
2. RLE/bit-packed dictionary indices for present positions;
3. values from the previously parsed `Dictionary`.

A missing dictionary for a dictionary encoding is a controlled `IOException`.
A bit width greater than 32 is rejected before index decoding.

There is also a synthetic path. `PageDecoder.nullPage(numValues)` creates an
all-null typed page without page bytes, decompression, or value decode. It is
used for optional columns when inline statistics prove a page cannot match.
It is forbidden for required columns because a required leaf cannot represent
the null placeholder used to preserve sibling-column alignment.

## Parquet format rules and Hardwood choices

### Format rules

- Page headers state type, compressed/uncompressed sizes, and type-specific
  headers.
- V1 compresses the full data-page body; repetition/definition streams inside
  the decompressed body are each prefixed by a four-byte length when their
  maximum level is nonzero.
- V2 stores level regions uncompressed and identifies their lengths in the
  header; only the value region may be compressed.
- Level streams use RLE/bit-packed hybrid encoding in the modern layouts
  handled here.
- Encoded values omit null leaves; definition levels determine where values
  belong.
- A page CRC, when present, protects the stored page body.
- The data page header's encoding selects how the value region is represented.

### Hardwood choices

- `PageDecoder` returns a sealed hierarchy backed by primitive arrays rather
  than boxed objects.
- Arrays use logical-position indexing. Null slots retain default primitive
  values and are guarded by definition levels/validity downstream.
- `null` definition levels mean all-present; this avoids level-array
  allocation and enables bulk-copy assembly.
- Level buffers are pooled by reorder slot through `LevelScratch`; `size()`
  fences stale tails.
- CRC is checked immediately after body slicing and before decompression.
- Decoder/type combinations not implemented fail explicitly rather than
  silently interpreting bytes using another encoding.
- The fixed-size-list fast path recognizes a narrow level geometry and stamps
  `fixedListK`; regular decoding remains the fallback.
- `nullPage` is an execution shortcut for optional filtered columns, not an
  on-disk page type.

## Current code tour

1. [`PageDecoder`](../../core/src/main/java/dev/hardwood/internal/reader/PageDecoder.java)
   is the main trace. Read `decodePage`, `parseDataPage`,
   `parseDataPageV2`, `readValueRegion`, level helpers, and
   `decodeTypedValues` in that order.
2. [`Page`](../../core/src/main/java/dev/hardwood/internal/reader/Page.java)
   defines the sealed typed records and the critical `size`/pooled-array
   contract.
3. [`CrcValidator`](../../core/src/main/java/dev/hardwood/internal/reader/CrcValidator.java)
   validates the stored body before decoding.
4. [`DecompressorFactory`](../../core/src/main/java/dev/hardwood/internal/compression/DecompressorFactory.java)
   maps the column codec to a decompressor.
5. [`PlainDecoder`](../../core/src/main/java/dev/hardwood/internal/encoding/PlainDecoder.java)
   shows how definition levels control dense-byte to sparse-slot placement.
6. [`RleBitPackingHybridDecoder`](../../core/src/main/java/dev/hardwood/internal/encoding/RleBitPackingHybridDecoder.java)
   decodes level streams and dictionary indices.
7. [`Dictionary`](../../core/src/main/java/dev/hardwood/internal/reader/Dictionary.java)
   creates typed pages from dictionary indices.
8. [`FixedSizeListDetector`](../../core/src/main/java/dev/hardwood/internal/reader/FixedSizeListDetector.java)
   recognizes the guarded list fast path.
9. [`ColumnWorker`](../../core/src/main/java/dev/hardwood/internal/reader/ColumnWorker.java)
   shows how decode tasks invoke the package-private overload with
   slot-owned `LevelScratch`.
10. [`PARSING_PIPELINE_V2`](../../_designs/PARSING_PIPELINE_V2.md) places page
    decoding on the shared bounded platform-thread pool.

Useful integration and regression tests:

- [`CrcValidationTest`](../../core/src/test/java/dev/hardwood/CrcValidationTest.java)
  corrupts data and dictionary bodies and expects attributed failure.
- [`ByteArrayDictionaryInternTest`](../../core/src/test/java/dev/hardwood/internal/reader/ByteArrayDictionaryInternTest.java)
  pins binary dictionary reuse.
- [`FixedSizeListEngagementTest`](../../core/src/test/java/dev/hardwood/internal/reader/FixedSizeListEngagementTest.java)
  checks when the fixed-list path engages and falls back.
- [`ParquetReaderTest`](../../core/src/test/java/dev/hardwood/ParquetReaderTest.java)
  supplies end-to-end physical/logical value checks.

## Guided lab: trace one page from bytes to an array

Run from the repository root.

### 1. Annotate the exact sequence

In `PageDecoder.decodePage`, number these operations in source order:

- parse header;
- measure header;
- slice body;
- CRC;
- select V1/V2;
- decompress appropriate region;
- decode levels;
- decode values;
- commit JFR event.

For each, write one sentence explaining what later step depends on it.

### 2. Hand-trace null placement

Use the worked optional-INT32 example. On paper, change definition levels to:

```text
[0, 1, 0, 1, 1, 0]
```

and encoded PLAIN values to:

```text
[7, 8, 9]
```

Write the six-slot `int[]`, identify which slots are semantically readable,
and state `Page.size()`. Then inspect `PlainDecoder.readInts` to confirm the
cursor behavior.

### 3. Compare V1 and V2 source paths

Make a two-column table:

```text
question                         | V1 | V2
where level lengths come from    |    |
whether level bytes are compressed|   |
region passed to decompressor    |    |
how valuesOffset is computed     |    |
```

Use source, not memory. A contributor-grade diagnosis should immediately
notice a bug that subtracts V2 level lengths incorrectly.

### 4. Run corruption tests on real fixtures

`CrcValidationTest` reads checked-in valid and deliberately corrupted fixture
bytes. Predict whether a corrupt compressed body should reach a value decoder.

```shell
timeout 180s ./mvnw -pl core -Dtest=CrcValidationTest test
```

Confirm the failure is attributed to the relevant column/file boundary and is
raised before bad bytes are trusted.

### 5. Exercise dictionary and fixed-list specializations

```shell
timeout 180s ./mvnw -pl core -Dtest=ByteArrayDictionaryInternTest,FixedSizeListEngagementTest test
```

For one dictionary-backed binary page, follow:

```text
index bit width -> index decoder -> Dictionary.decodePage
-> ByteArrayPage(dictionary, dictIndices)
```

For one fixed-list case, identify every gate required before
`Page.withFixedListK` is used. The optimized path must be narrower than the
format's valid input space.

### 6. Run an end-to-end sanity test

```shell
timeout 180s ./mvnw -pl core -Dtest=ParquetReaderTest test
```

Pick one assertion involving nulls or binary values. Trace backward to the
`Page` field that preserves the relevant information.

## Common misconceptions

**“Decompression decodes values.”**  
It only reconstructs encoded bytes. PLAIN, delta, dictionary, or another
decoder still has to interpret them.

**“`num_values` equals encoded primitive count.”**  
It equals level positions. Null positions have no encoded leaf value.

**“A zero in an `IntPage.values()` array means SQL zero.”**  
Only if the position is present. A null slot also contains Java's default
zero.

**“A V2 page body can be passed whole to the codec.”**  
V2 level regions are uncompressed prefixes. Only its value region may be
compressed.

**“`definitionLevels().length` is the page size.”**  
Scratch arrays can be longer than the current page. Iterate to `size()`.

**“Logical timestamps should be constructed inside `PageDecoder`.”**  
The decoder produces physical arrays. Logical interpretation belongs at a
later conversion/access boundary.

## Recap and debugging checklist

- [ ] Does the input buffer contain exactly one complete data page?
- [ ] Did header parsing yield a trustworthy `headerSize` and body size?
- [ ] Was CRC checked over the stored body before decompression?
- [ ] Did page type choose V1 or V2 body rules?
- [ ] For V1, were the level length prefixes read little-endian after whole-body
      decompression?
- [ ] For V2, were level prefixes left outside value decompression?
- [ ] Were repetition and definition streams decoded for exactly
      `num_values` positions?
- [ ] Does a `null` definition array intentionally mean all-present?
- [ ] Did value decode consume bytes only for present leaf positions?
- [ ] Is the encoding valid for the physical type and is a required dictionary
      available?
- [ ] Does downstream iteration use `Page.size()`, not pooled array length?
- [ ] Is logical conversion being debugged after, rather than inside, physical
      page decoding?

## Quiz

1. List the operations performed by `decodePage` before it branches on
   `DATA_PAGE` versus `DATA_PAGE_V2`.
2. Why must definition levels be known before decoding PLAIN values for an
   optional column?
3. For definition levels `[1,0,1,0,1]` and encoded INT32 values
   `[4,5,6]`, what are `values`, `size`, and the semantically readable
   indices?
4. A V2 body has 10 repetition-level bytes, 6 definition-level bytes, and 80
   stored value bytes. Its uncompressed page size is 136. If values are
   compressed, what expected uncompressed size is passed to the decompressor?
5. Why may `page.definitionLevels().length > page.size()`, and what bug can
   result from iterating to array length?
6. What two checks occur before dictionary indices can be decoded, and what
   happens if either fails?
7. Explain why `ByteArrayPage` is not backed by one primitive array like
   `IntPage`, and what extra information a dictionary-backed instance can
   retain.
8. Where should a regression test go if an LZ4-compressed V2 page incorrectly
   includes uncompressed level bytes in the codec input: codec unit test,
   `PageDecoder`-level test, or row assembly test? Defend the earliest useful
   boundary.

[Answer key](../answers/13-page-decoding.md)
