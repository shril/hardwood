# Chapter 9 — Delta, Byte-Stream Split, and Codecs

Allow 45–75 minutes. The goal is to choose the right reverse transformation
from metadata instead of guessing from bytes.

## Objectives

After this chapter you should be able to:

1. distinguish value encoding from page compression;
2. summarize the three delta encoding families;
3. hand-reassemble a BYTE_STREAM_SPLIT payload;
4. select a decoder from the data-page header and a decompressor from
   column-chunk metadata;
5. account for nulls when initializing dense decoders; and
6. trace Hardwood's current encoding and codec dispatch.

## Prerequisite recap

Chapter 5 established that physical type controls the kind of value being
decoded. Chapter 6 established V1 and V2 compression boundaries. Chapters 7
and 8 showed PLAIN and dictionary value encodings.

Do not collapse these metadata fields:

```text
physical type:       what one decoded value physically is
page encoding:       how a dense sequence of those values became bytes
compression codec:   how page-region bytes became fewer stored bytes
logical type:        how the decoded physical value should be interpreted
```

## Central mental model: two reversible transforms in a fixed order

On write:

```text
typed dense values
    | value encoding (PLAIN, DELTA_*, BYTE_STREAM_SPLIT, dictionary...)
    v
encoded value bytes
    | page compression (SNAPPY, GZIP, ZSTD...)
    v
stored bytes
```

On read, reverse the arrows:

```text
stored bytes --decompress--> encoded value bytes --decode--> typed values
```

**Parquet format rule:** the data-page header's `encoding` selects the value
decoder for that page. The column chunk's `codec` selects the decompressor.
The chunk's `encodings` list describes encodings used in the chunk but does not
replace each page's actual encoding field.

Compression is byte-to-byte and does not know whether the bytes contain
integers or strings. Encoding is value-aware and does not know whether its
result will later be compressed.

## Delta encoding families

### DELTA_BINARY_PACKED

This encoding targets `INT32` and `INT64` sequences. Its stream begins with:

```text
block size | miniblock count | total value count | first value
```

The first three fields are unsigned varints; the first value is zigzag-varint
encoded. For each block:

1. compute deltas between adjacent values;
2. store the minimum delta as a zigzag varint;
3. subtract that minimum from each delta;
4. store one bit width per miniblock; and
5. bit-pack the non-negative adjusted deltas.

For:

```text
100, 103, 107, 110
```

the deltas are:

```text
3, 4, 3
```

The minimum is 3, so adjusted deltas are:

```text
0, 1, 0
```

Those adjusted values need only one bit. Reconstruction adds the minimum back,
then cumulatively adds each delta to the previous value.

### DELTA_LENGTH_BYTE_ARRAY

For a sequence of `BYTE_ARRAY` values:

```text
delta-binary-packed lengths | concatenated raw bytes
```

For `"cat"`, `""`, `"table"`:

```text
lengths = 3, 0, 5
data    = "cattable"
```

The length decoder's final byte position tells the reader where concatenated
data starts.

### DELTA_BYTE_ARRAY

This is prefix compression for `BYTE_ARRAY` and
`FIXED_LEN_BYTE_ARRAY` sequences:

```text
delta-binary-packed prefix lengths
    |
    +-- followed by DELTA_LENGTH_BYTE_ARRAY suffixes
```

For `"apple"`, `"application"`, `"apply"`:

```text
prefix lengths: 0, 4, 4
suffixes:       "apple", "ication", "y"
```

Each value is reconstructed from a prefix of the previous value plus its
suffix. A bad prefix length therefore corrupts the reconstruction chain, not
just one isolated output.

**Parquet format rule:** all three delta streams operate over dense non-null
values. Null positions are supplied separately by definition levels.

## BYTE_STREAM_SPLIT

For `N` fixed-width values of `K` bytes, PLAIN would place each value's bytes
together. BYTE_STREAM_SPLIT transposes that `N × K` rectangle:

```text
stream 0: byte 0 from every value
stream 1: byte 1 from every value
...
stream K-1: byte K-1 from every value
```

The streams are concatenated. This often gives a compressor long runs of
similar exponent or high-order bytes for numeric data.

**Parquet format rule:** BYTE_STREAM_SPLIT is defined for fixed-width physical
types including `FLOAT`, `DOUBLE`, `INT32`, `INT64`, and
`FIXED_LEN_BYTE_ARRAY`. It is an encoding, not a compression codec.

## Hand-worked byte/value example

Use two four-byte physical words:

```text
value 0 = 0x11223344 -> little-endian bytes 44 33 22 11
value 1 = 0xA1B2C3D4 -> little-endian bytes D4 C3 B2 A1
```

Transpose by byte position:

```text
stream 0: 44 D4
stream 1: 33 C3
stream 2: 22 B2
stream 3: 11 A1
```

The encoded value region is:

```text
44 D4 33 C3 22 B2 11 A1
```

To reconstruct value `i`, gather:

```text
encoded[k * N + i], for k = 0..K-1
```

For `i = 1` and `N = 2`:

```text
k=0: encoded[1] = D4
k=1: encoded[3] = C3
k=2: encoded[5] = B2
k=3: encoded[7] = A1
```

The gathered little-endian bytes `D4 C3 B2 A1` recover `0xA1B2C3D4`.

If the logical page has three positions with one null, `N` is two, not three.
Using the level-entry count would choose incorrect stream starts.

## Compression dispatch and page version

For V1, read in this order:

```text
header -> CRC -> decompress whole body -> split levels/value -> decode value encoding
```

For V2:

```text
header -> CRC -> split raw levels/stored value region
       -> decompress values only when is_compressed
       -> decode value encoding
```

Dictionary pages also use the column chunk's codec. Their decompressed body is
then PLAIN-decoded into dictionary entries.

**Hardwood choice:** the current `DecompressorFactory` dispatches
`UNCOMPRESSED`, `GZIP`, `SNAPPY`, `ZSTD`, `LZ4`, `LZ4_RAW`, and `BROTLI`.
`LZO` is explicitly unsupported. Native/third-party codec implementations are
checked at runtime where required.

**Hardwood choice:** current data-page decoding supports:

| Encoding | Current physical-type path |
|---|---|
| `PLAIN` | all Hardwood physical types |
| `DELTA_BINARY_PACKED` | `INT32`, `INT64` |
| `DELTA_LENGTH_BYTE_ARRAY` | byte-array values |
| `DELTA_BYTE_ARRAY` | byte-array-backed values, including tested FLBA |
| `BYTE_STREAM_SPLIT` | `FLOAT`, `DOUBLE`, `INT32`, `INT64`, `FIXED_LEN_BYTE_ARRAY` |
| `RLE_DICTIONARY`, `PLAIN_DICTIONARY` | typed dictionary lookup for every physical type except `BOOLEAN` |
| `RLE` | Boolean values |

An enum constant's existence is not proof that every type/encoding pairing is
supported. In particular, Hardwood rejects Boolean dictionary construction
explicitly. The dispatch branch, dictionary factory, and tests together are
the evidence.

## Current Hardwood code tour

1. [`PageDecoder`](../../core/src/main/java/dev/hardwood/internal/reader/PageDecoder.java)
   first dispatches page version and codec boundaries, then switches on the
   page's value `Encoding` in `decodeTypedValues`.
2. [`DecompressorFactory`](../../core/src/main/java/dev/hardwood/internal/compression/DecompressorFactory.java)
   maps `CompressionCodec` to an implementation and rejects LZO.
3. [`DeltaBinaryPackedDecoder`](../../core/src/main/java/dev/hardwood/internal/encoding/DeltaBinaryPackedDecoder.java)
   reads stream and block headers, then reconstructs cumulative values.
4. [`DeltaLengthByteArrayDecoder`](../../core/src/main/java/dev/hardwood/internal/encoding/DeltaLengthByteArrayDecoder.java)
   initializes lengths for the non-null count and continues at the length
   decoder's final position.
5. [`DeltaByteArrayDecoder`](../../core/src/main/java/dev/hardwood/internal/encoding/DeltaByteArrayDecoder.java)
   composes a prefix decoder with a suffix decoder and retains the previous
   value.
6. [`ByteStreamSplitDecoder`](../../core/src/main/java/dev/hardwood/internal/encoding/ByteStreamSplitDecoder.java)
   computes stream offsets as `k * numValues` and gathers one byte from each.
7. [`DeltaBinaryPackedTest`](../../core/src/test/java/dev/hardwood/DeltaBinaryPackedTest.java)
   and
   [`DeltaByteArrayFlbaTest`](../../core/src/test/java/dev/hardwood/DeltaByteArrayFlbaTest.java)
   pin current dispatch combinations against checked-in fixtures.

Notice the dense-count calls in `PageDecoder` before BYTE_STREAM_SPLIT and both
byte-array delta decoders. Decoder initialization must use present values, not
`num_values`.

## Guided lab

### 1. Ask metadata two different questions

Use:

- [`delta_binary_packed_test.parquet`](../../core/src/test/resources/delta_binary_packed_test.parquet)
- [`plain_snappy.parquet`](../../core/src/test/resources/plain_snappy.parquet)

```shell
hardwood inspect pages -f core/src/test/resources/delta_binary_packed_test.parquet \
  --column value_i32
hardwood inspect columns -f core/src/test/resources/plain_snappy.parquet
hardwood inspect pages -f core/src/test/resources/plain_snappy.parquet
```

Write separate answers for:

1. Which value encoding does the data-page header select?
2. Which codec does the column chunk select?

The Snappy fixture intentionally combines PLAIN encoding with SNAPPY
compression, demonstrating that neither implies the other.

### 2. Run focused decoder paths

```shell
timeout 180s ./mvnw -pl core \
  -Dtest=DeltaBinaryPackedTest,DeltaByteArrayTest,DeltaByteArrayFlbaTest test

timeout 180s ./mvnw -pl core \
  -Dtest=ParquetReaderTest#testReadSnappyCompressedParquet test
```

For each test class, identify the assertion that proves the metadata selected
the intended encoding or codec and one assertion that proves decoded values.

### 3. Trace a dispatch

In
[`PageDecoder.decodeTypedValues`](../../core/src/main/java/dev/hardwood/internal/reader/PageDecoder.java),
trace an optional `BYTE_STREAM_SPLIT` `DOUBLE` page with:

```text
num_values = 10
definition levels contain 3 nulls
```

Record:

- the `numNonNullValues` passed to the decoder;
- the byte width selected;
- the output array length; and
- the number of values `gatherBytes` will consume.

Explain why the output array and encoded stream have different counts.

## Common misconceptions

- **“Encoding and compression are synonyms.”** Encoding is value-to-byte;
  compression is byte-to-byte.
- **“The chunk's encoding list tells me how this page is encoded.”** The
  page-header field is the per-page selector.
- **“BYTE_STREAM_SPLIT compresses floating-point values.”** It only transposes
  bytes; a codec may compress the result.
- **“Delta means every delta is stored directly.”** DELTA_BINARY_PACKED removes
  a block minimum and packs adjusted deltas.
- **“A decoder is initialized with `num_values`.”** Dense encodings use the
  number of maximum-definition positions.
- **“All codec enum values work in Hardwood.”** LZO is currently rejected.
- **“V2 level bytes go through the selected codec.”** They are always raw.

## Recap and debugging checklist

- [ ] Read physical type, page encoding, and chunk codec separately.
- [ ] Apply decompression before value decoding.
- [ ] Respect V1 whole-body and V2 value-only compression boundaries.
- [ ] Initialize dense decoders with the non-null count.
- [ ] Validate that the encoding supports the physical type.
- [ ] For delta byte arrays, preserve decoder hand-off positions.
- [ ] For BYTE_STREAM_SPLIT, use `streamOffset = k × denseCount`.
- [ ] Treat unsupported codec and unsupported encoding/type as different
      failures.

## Quiz

1. Explain the difference between `encoding = PLAIN` and `codec = SNAPPY` on
   one page.
2. For values `100, 103, 107, 110`, calculate deltas, minimum delta, and
   adjusted deltas.
3. BYTE_STREAM_SPLIT bytes for two four-byte values are
   `01 05 02 06 03 07 04 08`. Reconstruct both little-endian words.
4. A page has 12 level entries, 5 nulls, and BYTE_STREAM_SPLIT `DOUBLE`
   values. What `N`, `K`, and uncompressed value byte count should the decoder
   use?
5. Which metadata selects the decompressor, and which selects the value
   decoder?
6. Why must a V1 reader decompress before reading level boundaries?
7. A compressed V2 body has 4 repetition bytes, 5 definition bytes, and 21
   stored value bytes. `uncompressed_page_size` is 73. What input and expected
   output lengths go to the decompressor?
8. What happens in current Hardwood if a page declares
   `DELTA_BINARY_PACKED` for physical `DOUBLE`, even when decompression
   succeeds?

Compare your work with the
[answer key](../answers/09-encodings-and-codecs.md).
