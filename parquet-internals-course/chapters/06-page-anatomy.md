# Chapter 6 — Pages, Headers, Compression, and CRC

Allow 45–75 minutes. By the end, a page should look like a bounded envelope,
not an opaque byte range.

## Objectives

After this chapter you should be able to:

1. separate a serialized `PageHeader` from the page body;
2. use `compressed_page_size` and `uncompressed_page_size` without counting
   the header;
3. draw the body boundaries of `DATA_PAGE` (V1) and `DATA_PAGE_V2`;
4. state which V2 regions can be compressed;
5. explain what an optional page CRC covers; and
6. trace those rules through Hardwood's current `PageDecoder`.

## Prerequisite recap

Chapters 1–5 established that a column chunk contains pages, integer payloads
often use little-endian order, and metadata structures use Thrift Compact
Protocol. Keep three facts active:

- a page belongs to one leaf column and therefore has one physical type;
- the page header is a Thrift structure whose serialized length is not fixed;
- schema repetition determines whether repetition and definition level streams
  are needed.

## Central mental model: an envelope followed by measured compartments

A page is a variable-length header followed immediately by a body:

```text
page start
    |
    v
+----------------------+-------------------------------+
| Thrift PageHeader    | page body                     |
| variable byte length | compressed_page_size bytes    |
+----------------------+-------------------------------+
                       ^
                       body start = page start + bytes read for header
```

The header tells you how many body bytes to take. It does **not** tell you its
own serialized length; the Thrift reader's final position does.

**Parquet format rule:** `PageHeader.compressed_page_size` and
`uncompressed_page_size` describe the body only. The next page begins at:

```text
page start + serialized header size + compressed_page_size
```

### Data Page V1

After decompression, a V1 data-page body has these regions:

```text
+--------------------------+--------------------------+----------------+
| repetition levels        | definition levels        | encoded values |
| if max repetition > 0    | if max definition > 0    | dense stream   |
+--------------------------+--------------------------+----------------+
```

With the usual RLE level encoding, each present V1 level region starts with a
four-byte little-endian byte length, followed by exactly that many hybrid
RLE/bit-packed bytes. Those four bytes are inside the page body. A required
flat column has neither level region.

**Parquet format rule:** the codec named in column-chunk metadata applies to
the V1 body as one unit. A reader cannot locate compressed V1 subregions by
looking at the stored bytes; it first decompresses the whole body.

### Data Page V2

V2 puts the level byte lengths in `DataPageHeaderV2`:

```text
+---------------------+---------------------+--------------------------+
| repetition levels   | definition levels   | encoded values           |
| rep byte length     | def byte length     | remaining body bytes     |
| never compressed    | never compressed    | maybe compressed         |
+---------------------+---------------------+--------------------------+
```

There are no four-byte length prefixes in the V2 body. The body starts with
exactly `repetition_levels_byte_length` bytes, then exactly
`definition_levels_byte_length` bytes. `is_compressed` controls only the value
region and defaults to `true` when omitted from the Thrift header.

**Parquet format rule:** for V2:

```text
compressed_page_size
  = raw repetition bytes + raw definition bytes + stored value bytes

uncompressed_page_size
  = raw repetition bytes + raw definition bytes + uncompressed value bytes
```

Thus the word “compressed” in `compressed_page_size` does not mean every byte
in that count passed through a codec.

### CRC

`PageHeader.crc` is optional. When present, it is a CRC-32 of the page body as
stored: exactly `compressed_page_size` bytes immediately after the header.
For V2 that includes the raw level regions plus the stored value region.

**Parquet format rule:** the header itself is not covered by the page CRC.

**Hardwood choice:** Hardwood validates a present CRC before decompression and
throws an `IOException` on mismatch. A missing CRC is accepted; CRC emission is
optional in the format.

## Hand-worked page body

Consider a flat optional `INT32` column with logical positions:

```text
[10, null, 20]
```

Its maximum definition level is 1. There is no repetition stream. The
definition levels are `[1, 0, 1]`, while the dense value stream contains only
`[10, 20]`.

Suppose the definition levels are represented by one RLE run of three `1`
values only for illustration? That would be wrong for the middle null. Instead,
use a bit-packed hybrid group padded to eight values:

```text
levels:         1 0 1 0 0 0 0 0
bit width:      1
packed header:  03          # (one group << 1) | 1
packed byte:    05          # bits, least-significant first: 1,0,1,0,0,0,0,0
hybrid bytes:   03 05
```

The PLAIN values are little-endian `INT32`s:

```text
10 -> 0A 00 00 00
20 -> 14 00 00 00
```

A V1 uncompressed body is therefore:

```text
02 00 00 00  03 05  0A 00 00 00  14 00 00 00
^^^^^^^^^^^   ^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^^
def byte len  defs   two dense PLAIN values
```

The body is 14 bytes. Therefore both page sizes are 14 when the column codec is
`UNCOMPRESSED`. If the serialized `PageHeader` occupies 19 bytes and starts at
file offset 1,000:

```text
header: [1000, 1019)
body:   [1019, 1033)
next:   1033
```

The V2 uncompressed body for the same positions is just:

```text
03 05  0A 00 00 00  14 00 00 00
^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^^
defs   values
```

Its V2 header says `repetition_levels_byte_length = 0`,
`definition_levels_byte_length = 2`, `num_values = 3`,
`num_nulls = 1`, and `num_rows = 3`. Both page sizes are 10 if
`is_compressed = false`. Notice what disappeared: only the V1 four-byte level
length prefix.

Now take a compressed V2 body with two repetition bytes, three definition
bytes, and ten stored value bytes that expand to twenty:

```text
compressed_page_size   = 2 + 3 + 10 = 15
uncompressed_page_size = 2 + 3 + 20 = 25
```

The decompressor receives the last 10 bytes and is asked for 20 bytes. Passing
all 15 body bytes to it would incorrectly treat level bytes as codec input.

## Format rules and Hardwood choices

**Parquet format rule**

- Page headers use Thrift Compact Protocol and have variable length.
- The body begins where header parsing stops.
- V1 compresses its whole body; V2 never compresses levels.
- V1 RLE level sections have four-byte body length prefixes; V2 level lengths
  live in the header and have no body prefixes.
- A page CRC, when present, covers the stored body and not the header.

**Hardwood choice**

- Hardwood slices a `ByteBuffer` for the body instead of copying it first.
- It validates CRC on that slice before any decompression.
- It decompresses V1 in one operation and V2 values in a separate operation.
- It materializes level bytes into arrays when needed and can represent an
  all-present definition stream without an `int[]`; later chapters revisit
  that representation.

## Current Hardwood code tour

Follow these links in order:

1. [`PageHeaderReader`](../../core/src/main/java/dev/hardwood/internal/thrift/PageHeaderReader.java)
   reads fields 1–8 and leaves the enclosing `ThriftCompactReader` positioned
   after the header.
2. [`DataPageHeaderReader`](../../core/src/main/java/dev/hardwood/internal/thrift/DataPageHeaderReader.java)
   exposes the V1 value and level encodings.
3. [`DataPageHeaderV2Reader`](../../core/src/main/java/dev/hardwood/internal/thrift/DataPageHeaderV2Reader.java)
   reads `num_rows`, both level byte lengths, and the default-true
   `is_compressed`.
4. [`PageDecoder`](../../core/src/main/java/dev/hardwood/internal/reader/PageDecoder.java)
   uses `getBytesRead()` as `headerSize`, slices exactly
   `compressedPageSize()` bytes, checks CRC, and dispatches to
   `parseDataPage` or `parseDataPageV2`.
5. In that same file, compare `parseDataPage`, which reads little-endian
   four-byte level lengths after whole-body decompression, with
   `readValueRegion`, which excludes V2's raw levels from decompression.
6. [`CrcValidator`](../../core/src/main/java/dev/hardwood/internal/reader/CrcValidator.java)
   computes CRC-32 over a duplicate buffer so validation does not move the
   caller's position.

The current page-delivery path is `RowGroupIterator` → `PageSource` →
`ColumnWorker` → `PageDecoder`. Do not use old architecture descriptions as a
class-name index.

## Guided lab

Run commands from the repository root.

### 1. Compare page kinds

With a built CLI on `PATH`, inspect a small V1 fixture and a V2 nested fixture:

```shell
hardwood inspect pages -f core/src/test/resources/plain_uncompressed.parquet
hardwood inspect pages -f core/src/test/resources/misaligned_pages_nested_v2.parquet
```

Record each displayed page type, encoding, compressed size, and value count.
The command deliberately reports the body size from the header, not
header-plus-body.

Fixtures:

- [`plain_uncompressed.parquet`](../../core/src/test/resources/plain_uncompressed.parquet)
- [`misaligned_pages_nested_v2.parquet`](../../core/src/test/resources/misaligned_pages_nested_v2.parquet)

### 2. Trace the two branches

Open
[`PageDecoder.decodePage`](../../core/src/main/java/dev/hardwood/internal/reader/PageDecoder.java)
and answer:

1. At what statement does the body start become known?
2. Is CRC checked before or after the `switch` on page type?
3. Which branch asks the decompressor for the full
   `uncompressed_page_size`?
4. Which branch subtracts level lengths before choosing the expected
   decompressed value length?

### 3. Exercise CRC behavior

```shell
timeout 180s ./mvnw -pl core -Dtest=CrcValidationTest test
```

Read
[`CrcValidationTest`](../../core/src/test/java/dev/hardwood/CrcValidationTest.java)
before running it. Identify one test for a valid data page, one for a corrupt
data page, and the equivalent pair for a dictionary page. Explain why byte
corruption must be detected before a decoder interprets values.

## Common misconceptions

- **“Page size includes the header.”** It does not. Add the serialized header
  length separately when advancing.
- **“V2 pages are entirely uncompressed.”** Only V2 level regions are always
  raw; the value region may be compressed.
- **“V2 repeats V1's four-byte level lengths.”** It does not; lengths are
  Thrift fields.
- **“`num_values` means dense non-null values.”** It counts level positions,
  including null and empty-container positions. Chapter 7 makes this explicit.
- **“CRC protects the decoded values.”** It protects stored body bytes. It
  catches corruption before decoding but does not prove semantic correctness.
- **“A missing CRC is corrupt.”** CRC is optional.

## Recap and debugging checklist

When a page boundary or decompression failure appears, check:

- [ ] Did header parsing stop at the true body start?
- [ ] Did you slice exactly `compressed_page_size` body bytes?
- [ ] Did you exclude the header from both page-size arithmetic and CRC?
- [ ] For V1, did you decompress before reading level lengths?
- [ ] For V2, did you use header level lengths without four-byte prefixes?
- [ ] For V2, did only the stored value region enter the decompressor?
- [ ] Is expected V2 value output
      `uncompressed_page_size - rep_length - def_length`?
- [ ] Did page advancement include both header and stored body lengths?

## Quiz

1. A page starts at offset 400. Its serialized header is 17 bytes and
   `compressed_page_size` is 83. What is the next page offset?
2. A V2 body has 4 repetition bytes, 6 definition bytes, and 30 compressed
   value bytes that expand to 90. Compute both page-size fields.
3. Why can a V1 reader not inspect compressed body bytes to locate the value
   region?
4. Where is an RLE definition stream's byte length stored in V1, and where is
   it stored in V2?
5. Which exact bytes does a page CRC cover?
6. In the hand-worked optional `INT32` example, why are there three definition
   entries but only eight PLAIN value bytes?
7. Trace the failure if a reader passes an entire V2 body, including five raw
   level bytes, to the codec and requests only the uncompressed value length.
8. In `PageDecoder`, what evidence determines the variable header size, and
   what metadata determines the following body size?

Compare your work with the
[answer key](../answers/06-page-anatomy.md).
