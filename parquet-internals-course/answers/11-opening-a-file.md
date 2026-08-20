# Answers — Chapter 11: Opening a File in Hardwood

[Return to the lesson](../chapters/11-opening-a-file.md)

## 1. `InputFile` read contract

The four methods are:

1. `open()` — acquire any expensive backend resources;
2. `length()` — report the opened file's byte length;
3. `readRange(long offset, int length)` — return the requested bytes;
4. `name()` — identify the source in errors, logs, and events.

`close()` completes the lifecycle, but the question excludes it. After
`open()`, implementations must support concurrent calls from the reader
pipeline.

## 2. Footer range calculation

```text
footerStart = 20,000 - 8 - 1,200 = 18,792
footerEnd   = 20,000 - 8         = 19,992
```

The footer body occupies half-open range `[18,792, 19,992)`. Bytes
`[19,992, 19,996)` are the length and `[19,996, 20,000)` are final magic.

## 3. Why validate start magic separately?

The format requires both boundary magic values. Final `PAR1` plus a plausible
length does not prove byte 0 begins an ordinary Parquet file. The separate
read rejects a wrong prefix, identifies encrypted `PARE`, and keeps the error
at the metadata boundary rather than trusting arbitrary leading bytes.

This is Hardwood's validation strategy over the format's two-magic invariant;
the tail read cannot validate bytes it did not fetch.

## 4. Failure during public open

`openInternal` has already called `first.open()`. Metadata parsing occurs
inside a `try`. If it throws, the catch closes the first `InputFile` and
rethrows the original failure. If close itself fails, that close exception is
attached as suppressed rather than replacing the more useful opening error.

Thus a failed `ParquetFileReader` construction does not transfer a
half-opened file to a nonexistent reader.

## 5. Eager and cached multi-file metadata

For `openAll([a, b, c])`, `a` is opened eagerly. Its `FileMetaData` and
`FileSchema` are available immediately and seed cache entry 0.

`b` and `c` are prepared only on demand or prefetch. `FileMetadataCache`
stores a `CompletableFuture<PreparedFile>` by file index, so all access to `b`
within the parent reader joins the same load and footer parse. Accessing `b`
does not imply accessing `c`.

## 6. Metadata parsing versus schema construction

`FileMetaDataReader` validates and materializes the serialized Thrift
structure: field types, counts, enums, offsets, lists, and records.

`FileSchema.fromSchemaElements` interprets one part of that valid structure as
a Parquet schema tree and Hardwood leaf model. A footer can be readable
Thrift yet contain an invalid schema hierarchy. Keeping the boundaries
separate gives each failure an accurate layer and prevents later projection
code from seeing malformed schema state.

## 7. Custom backend obligations

After `open()`, the implementation must:

- make `length()` and `readRange()` safe under concurrent calls;
- return exactly the requested range or throw a controlled exception;
- enforce invalid offset/length bounds rather than return wrong bytes;
- return caller-owned `ByteBuffer` views whose position/content will not be
  unexpectedly changed by another request;
- keep the backing storage valid until close and coordinate close with the
  framework's lifecycle;
- provide a stable, human-readable `name()`;
- surface I/O failures rather than silently truncate or substitute data.

A slice of a shared cache is valid if its bytes remain stable for the caller's
use and concurrent requests do not mutate its view state.

## 8. Placement for a negative data-page offset regression

Start with the real-file metadata/public opening boundary represented by
`MalformedMetadataFileTest`, backed by a reproducible corrupt fixture. The
negative offset originates in footer metadata and should be rejected before
it can become an I/O plan.

A decoder unit test is too late: page decoding assumes planning has already
located a valid complete page. Letting the invalid offset reach that layer can
produce a misleading range or EOF error and loses the opportunity to attach
the precise file/metadata context.
