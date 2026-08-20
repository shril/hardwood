# Chapter 11 — Opening a File in Hardwood

**Study time:** 45–75 minutes

Chapters 1–10 built the format from bytes upward. This chapter starts the
contributor-level code trace: how a public Hardwood call turns an unopened
`InputFile` into trusted file metadata and a validated schema.

## Objectives

By the end of this lesson, you should be able to:

1. trace `ParquetFileReader.open(...)` through opening, tail reads, Thrift
   footer parsing, and `FileSchema` construction;
2. state the lifecycle and concurrency contract of `InputFile`;
3. calculate the footer body's byte range from the file length and final eight
   bytes;
4. distinguish first-file eager work from later-file lazy metadata loading;
5. identify the test boundary for malformed magic, footer, schema, and
   multi-file lifecycle failures.

## Prerequisite recap

Keep four facts from earlier chapters in working memory:

- A Parquet file begins and ends with four magic bytes. Ordinary files use
  `PAR1`; encrypted-footer files can use `PARE`.
- Immediately before the final magic is a four-byte little-endian footer-body
  length.
- The footer body is Thrift Compact Protocol `FileMetaData`. It describes the
  schema and row groups; it does not contain the column values themselves.
- Parquet stores primitive leaf columns. Hardwood derives a navigable
  `FileSchema` from the footer's flat depth-first list of schema elements.

The final layout is:

```text
offset 0                                                   fileSize
  |                                                           |
  v                                                           v
+------+------------------ data ------------------+--------+----+------+
| PAR1 | row groups / column chunks / pages       | footer |len | PAR1 |
+------+-------------------------------------------+--------+----+------+
                                                     ^       4     4
                                                     |
                                              footerStart
```

## Central mental model: opening establishes trust

Opening is not “start reading rows.” It establishes enough trusted,
projection-independent state to plan later reads:

```text
caller
  |
  | InputFile.of(path)                 creates an unopened adapter
  v
ParquetFileReader.open(inputFile)
  |
  +-- InputFile.open()                 acquire mapping/backend resources
  |
  +-- ParquetMetadataReader
  |     +-- length()
  |     +-- readRange(0, 4)            validate start magic
  |     +-- readRange(fileSize-8, 8)   footer length + end magic
  |     +-- validate footerStart
  |     +-- readRange(footerStart, n)  footer body
  |     `-- Thrift readers -> FileMetaData
  |
  +-- FileSchema.fromSchemaElements(...)
  |
  `-- ParquetFileReader               owns files and, sometimes, context
```

The boundary matters. Before metadata parsing, offsets and counts are
untrusted bytes. After controlled validation and schema construction, later
components can reason in terms of row groups, columns, codecs, and page
locations. A corrupt footer must fail here with the file name attached; it
must not leak into page fetching as a mysterious out-of-range request.

`InputFile` is deliberately a range-read abstraction, not a stream. The
reader needs the tail before most of the middle, and object-store
implementations need to translate the same calls into ranged requests.

## Worked control/data-flow example

Suppose `orders.parquet` is 50,000 bytes long. Its last eight bytes decode as:

```text
bytes 49,992..49,995: 00 10 00 00
bytes 49,996..49,999: 50 41 52 31  ("PAR1")
```

Because the length is little-endian:

```text
footerLength = 0x00001000 = 4096
footerInfoPos = 50,000 - 4 - 4 = 49,992
footerStart   = 50,000 - 4 - 4 - 4,096 = 45,896
```

The live control flow is:

1. The caller creates `InputFile.of(path)`. This selects
   `MappedInputFile`, but does not open it.
2. `ParquetFileReader.open(inputFile)` delegates to the one-element
   `openAll(...)` path.
3. `openInternal(...)` copies the input list, opens the first file, and invokes
   `ParquetMetadataReader.readMetadata(first)`.
4. The metadata reader rejects any file smaller than 12 bytes: start magic
   (4), footer length (4), and end magic (4).
5. It reads and validates the first four bytes.
6. It reads bytes `[49,992, 50,000)`, sets the `ByteBuffer` to little-endian,
   obtains `4096`, and validates the end magic.
7. It verifies that `footerStart >= 4`. This prevents a corrupt length from
   placing the footer before the opening magic.
8. It reads `[45,896, 49,992)` and passes that exact slice to
   `ThriftCompactReader` and `FileMetaDataReader`.
9. `FileSchema.fromSchemaElements(...)` turns the metadata schema elements
   into Hardwood's schema model.
10. A `FileOpenedEvent` records file name, size, row-group count, and column
    count. The resulting `ParquetFileReader` seeds `FileMetadataCache` with
    this first footer, so child readers do not parse it again.

If step 8 reports a negative count or an invalid enum, the failure remains an
opening/metadata failure. `ExceptionContext` adds `orders.parquet` so a
multi-file read remains attributable.

### The multi-file variation

For `openAll(List.of(part0, part1, part2))`, `part0` follows all ten steps
eagerly. `part1` and `part2` remain unopened while the caller only uses the
parent reader's first-file metadata. An indexed metadata request or child
reader construction admits later loads. `FileMetadataCache` stores one
`CompletableFuture<PreparedFile>` per admitted file load:

```text
part0: completed future seeded by ParquetFileReader
part1: absent until get/prefetch -> async open + footer + schema
part2: absent until get/prefetch -> async open + footer + schema
```

The cache shares successes and failures within that parent reader. Reopening a
new parent creates a new cache and may retry a failed backend. Each
`PreparedFile` contains that file's `InputFile`, metadata, schema, and row
groups. Schema compatibility against the first file is performed later by the
row-group pipeline. In the current implementation, child-pipeline
initialization builds its work list across the files it may need and waits for
their prepared metadata. Do not infer from the asynchronous cache type that
next-file footer parsing necessarily overlaps current-file page decoding.

## Parquet format rules and Hardwood choices

### Format rules

- Ordinary Parquet files carry matching `PAR1` magic at the beginning and
  end.
- The four bytes before the final magic encode the footer-body length as a
  little-endian 32-bit integer.
- The footer body is serialized `FileMetaData` using Thrift Compact Protocol.
- Schema elements, row groups, column chunks, codecs, offsets, and optional
  indexes are represented by the Parquet metadata model.
- Parquet Modular Encryption defines encrypted-footer and plaintext-footer
  modes; an implementation must not interpret encrypted content as ordinary
  metadata.

### Hardwood choices

- `InputFile` exposes `open`, `length`, `readRange`, `name`, and `close`
  instead of requiring seekable streams.
- The local-path factory uses memory mapping; the in-memory factory returns
  slices of a supplied `ByteBuffer`.
- Implementations must be safe for concurrent use after `open()`. Returned
  buffers belong to the caller and may be zero-copy slices.
- Hardwood performs a separate start-magic read as well as the tail and footer
  reads. This gives an early, explicit format check.
- `ParquetFileReader` opens the first file eagerly, lazily opens later files,
  and owns every supplied `InputFile`.
- A dedicated context is created by convenience overloads; overloads that
  receive a `HardwoodContext` leave that context owned by the caller.
- Unsupported encryption is rejected with a specific attributed exception.
- Metadata and schema for the first file are cached and exposed before any
  row reader or column reader is created.

Do not confuse an efficient implementation choice with a format guarantee.
For example, Parquet permits a remote reader, but it does not prescribe
`readRange`. Conversely, memory mapping is useful locally but is not part of
the file format.

## Current code tour

Follow these files in order. All links point at the current checkout.

1. [`InputFile`](../../core/src/main/java/dev/hardwood/InputFile.java) defines
   the backend contract and factories. Notice that `open()` precedes
   `length()` and `readRange()`, and that concurrent access after opening is
   required.
2. [`MappedInputFile`](../../core/src/main/java/dev/hardwood/internal/reader/MappedInputFile.java)
   implements local path reads. Compare its small-file mapping with its
   region-on-demand behavior for files larger than one mapping can represent.
3. [`ByteBufferInputFile`](../../core/src/main/java/dev/hardwood/internal/reader/ByteBufferInputFile.java)
   is the zero-copy in-memory implementation.
4. [`ParquetFileReader`](../../core/src/main/java/dev/hardwood/reader/ParquetFileReader.java)
   contains `open`, `openAll`, and `openInternal`. Trace ownership flags,
   cleanup on failed opening, first metadata parsing, schema construction,
   and cache seeding.
5. [`ParquetMetadataReader`](../../core/src/main/java/dev/hardwood/internal/reader/ParquetMetadataReader.java)
   is the exact magic/tail/footer algorithm. Record each checked arithmetic
   invariant and each `FetchReason`.
6. [`FileMetaDataReader`](../../core/src/main/java/dev/hardwood/internal/thrift/FileMetaDataReader.java)
   maps Thrift fields into Hardwood metadata records.
7. [`FileSchema`](../../core/src/main/java/dev/hardwood/schema/FileSchema.java)
   turns schema elements into the tree/leaf model consumed by projection and
   decoding.
8. [`FileMetadataCache`](../../core/src/main/java/dev/hardwood/internal/reader/FileMetadataCache.java)
   owns lazy per-file preparation. Study `registerLoad`, exception unwrapping,
   and close synchronization.
9. [`PARSING_PIPELINE_V2`](../../_designs/PARSING_PIPELINE_V2.md) explains the
   intended post-open architecture. Current source and tests override stale
   details in that completed design, including when later-file metadata is
   forced.

Relevant tests:

- [`EncryptedFileTest`](../../core/src/test/java/dev/hardwood/EncryptedFileTest.java)
  pins controlled rejection of encryption modes.
- [`MalformedMetadataFileTest`](../../core/src/test/java/dev/hardwood/MalformedMetadataFileTest.java)
  checks a corrupt footer-derived offset and file attribution.
- [`FileMetadataCacheTest`](../../core/src/test/java/dev/hardwood/internal/reader/FileMetadataCacheTest.java)
  exercises lazy metadata cache behavior.
- [`MultiFileRowReaderTest`](../../core/src/test/java/dev/hardwood/MultiFileRowReaderTest.java)
  checks lazy footer reads, sharing, retries, and ownership.
- [`SchemaCompatibilityTest`](../../core/src/test/java/dev/hardwood/SchemaCompatibilityTest.java)
  shows that parsing a schema and accepting it into a multi-file relation are
  separate steps.

## Guided lab: prove the opening sequence

Run from the repository root.

### 1. Inspect a real fixture's tail

Use the small checked-in fixture
`core/src/test/resources/plain_uncompressed.parquet`. The following Python
snippet only reads bytes; it does not regenerate the fixture:

```shell
.docker-venv/bin/python - <<'PY'
from pathlib import Path

p = Path("core/src/test/resources/plain_uncompressed.parquet")
data = p.read_bytes()
footer_length = int.from_bytes(data[-8:-4], "little")
footer_start = len(data) - 8 - footer_length
print("size:", len(data))
print("start magic:", data[:4])
print("footer length:", footer_length)
print("footer start:", footer_start)
print("end magic:", data[-4:])
assert data[:4] == b"PAR1"
assert data[-4:] == b"PAR1"
assert footer_start >= 4
PY
```

On paper, recompute `footerStart` from the printed values. Then find the same
subtraction in `ParquetMetadataReader`.

### 2. Trace public construction

In `ParquetFileReader`, locate:

- the single-file delegation to `openAll`;
- first-file `open()`;
- `readMetadata`;
- `FileSchema.fromSchemaElements`;
- cleanup when any of those operations fails;
- the constructor call that seeds `FileMetadataCache`.

Write down who owns the input file and context for:

1. `ParquetFileReader.open(file)`;
2. `ParquetFileReader.open(file, context)`.

### 3. Observe eager and lazy footer behavior

Read `MultiFileRowReaderTest.exposesPerFileMetadataLazily`. Predict each
`footerReadCount()` assertion before running it.

```shell
timeout 180s ./mvnw -pl core -Dtest=MultiFileRowReaderTest#exposesPerFileMetadataLazily test
```

Explain why metadata index `0` is returned by identity and why asking for
index `1` does not touch index `2`.

### 4. Exercise controlled failures

```shell
timeout 180s ./mvnw -pl core -Dtest=EncryptedFileTest,MalformedMetadataFileTest test
```

For one assertion from each test, identify the earliest layer that has enough
information to reject the file. Confirm that the expected message carries
file context.

### 5. Check the project-wide build contract

You do not need to run the full build for this reading lab. For a real
contribution, the required verification command is:

```shell
timeout 180s ./mvnw clean verify
```

## Common misconceptions

**“Opening the parent reader reads every file and every row group.”**
Only the first file is opened and parsed by `openAll`. Later metadata is
forced by indexed metadata access or child-pipeline initialization. Neither
parent nor child initialization fetches data pages merely to parse footers.

**“The footer length includes its own four bytes and the final magic.”**  
It describes the serialized footer body only. Hardwood subtracts
`footerLength + 4 + 4`.

**“`InputFile.of(path)` has already acquired the file.”**  
It creates an unopened adapter. The framework calls `open()`.

**“A successfully parsed second footer is automatically schema-compatible.”**  
Parsing establishes that the metadata is structurally readable. The
row-group pipeline separately validates touched columns against the reference
schema.

**“Child `RowReader.close()` should close the supplied files.”**  
The parent `ParquetFileReader` owns the `InputFile` lifecycle. Child readers
release pipeline state; parent close releases files and any owned context.

## Recap and debugging checklist

When an opening failure appears, check in this order:

- [ ] Was `InputFile.open()` called, and did it acquire the backend resource?
- [ ] Does `length()` report at least 12 bytes?
- [ ] Are both magic values valid, and is encryption being rejected explicitly?
- [ ] Was the footer length read little-endian?
- [ ] Does `footerStart = fileSize - 8 - footerLength` remain at or after byte 4?
- [ ] Does the footer range contain exactly the serialized metadata body?
- [ ] Did Thrift parsing fail on malformed metadata, with the input name attached?
- [ ] Did `FileSchema.fromSchemaElements` reject an invalid schema?
- [ ] In a multi-file read, was the failure opening, parsing, or compatibility?
- [ ] Is resource ownership at the parent reader rather than a child reader?

The key trace is:

```text
InputFile -> ParquetMetadataReader -> FileMetaData -> FileSchema
          -> FileMetadataCache -> ParquetFileReader
```

## Quiz

1. What four methods form the essential opened-file read contract of
   `InputFile`, excluding `close()` and the static factories?
2. A file is 20,000 bytes long and its final length field is 1,200. What byte
   range contains the footer body?
3. Why does `ParquetMetadataReader` read the first magic separately when the
   tail already contains an end magic?
4. Trace what happens to the first `InputFile` if footer parsing throws during
   `ParquetFileReader.open(...)`.
5. In `openAll([a, b, c])`, which metadata is available eagerly, and what
   object prevents repeated footer parsing for `b`?
6. Explain why `FileMetaData` parsing and `FileSchema` construction are two
   distinct failure boundaries.
7. A custom object-store `InputFile` returns buffers backed by a shared cache.
   Which contract statements must it satisfy for Hardwood's later concurrent
   pipeline to be safe?
8. Where would you first place a regression test for a negative
   `data_page_offset` parsed from a real corrupt fixture, and why is a decoder
   unit test too late?

[Answer key](../answers/11-opening-a-file.md)
