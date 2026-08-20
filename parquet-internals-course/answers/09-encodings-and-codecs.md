# Answers — Chapter 9: Delta, Byte-Stream Split, and Codecs

Return to the [lesson](../chapters/09-encodings-and-codecs.md).

## 1. PLAIN plus SNAPPY

`PLAIN` says how typed dense values become uncompressed value bytes. For
example, an `INT64` becomes eight little-endian bytes.

`SNAPPY` says how the relevant page byte region is compressed for storage.
Reading reverses both independent transforms:

```text
stored Snappy bytes -> uncompressed PLAIN bytes -> typed values
```

Changing SNAPPY to GZIP would not change PLAIN layout. Changing PLAIN to a
delta encoding would not by itself choose a different codec.

## 2. Delta calculation

Adjacent subtraction gives:

```text
103 - 100 = 3
107 - 103 = 4
110 - 107 = 3
```

So:

```text
deltas = [3, 4, 3]
minimum delta = 3
adjusted = [3-3, 4-3, 3-3] = [0, 1, 0]
```

The adjusted maximum is 1, requiring one bit. Reconstruction starts at 100,
adds `3+0`, then `3+1`, then `3+0`.

## 3. BYTE_STREAM_SPLIT reconstruction

There are `N = 2` values and `K = 4` streams:

```text
stream 0: 01 05
stream 1: 02 06
stream 2: 03 07
stream 3: 04 08
```

Gather index 0 from every stream:

```text
01 02 03 04 -> little-endian word 0x04030201
```

Gather index 1:

```text
05 06 07 08 -> little-endian word 0x08070605
```

## 4. Dense DOUBLE dimensions

The dense count is:

```text
N = 12 positions - 5 nulls = 7 values
```

A `DOUBLE` has:

```text
K = 8 bytes/value
```

Therefore the uncompressed BYTE_STREAM_SPLIT region is:

```text
N × K = 7 × 8 = 56 bytes
```

The logical output can still have 12 slots; definitions decide which seven
slots receive gathered values.

## 5. Dispatch metadata

`ColumnMetaData.codec` selects the decompressor for the column chunk. The
specific `DataPageHeader.encoding` or `DataPageHeaderV2.encoding` selects the
value decoder for that page.

The column metadata's `encodings` collection is an inventory, useful for
inspection and validation, but it is not the per-page dispatch field.

## 6. V1 ordering

V1 compresses the body as one region. The level lengths and streams are inside
that compressed region, so their apparent stored offsets have no relation to
the uncompressed layout. The reader must:

1. decompress the entire body;
2. read the little-endian level lengths from the recovered bytes;
3. locate the value region; and
4. invoke the value decoder.

## 7. V2 decompressor lengths

Raw levels occupy:

```text
4 + 5 = 9 bytes
```

They are excluded from codec input, leaving the stated **21 input bytes**.
The uncompressed size includes levels, so expected value output is:

```text
73 - 9 = 64 bytes
```

The call is conceptually:

```text
decompress(21 stored value bytes, expected output 64 bytes)
```

## 8. DELTA_BINARY_PACKED with DOUBLE

Decompression can succeed because the codec only sees bytes. Dispatch then
enters the `DELTA_BINARY_PACKED` branch, whose current type switch supports
only `INT32` and `INT64`. For `DOUBLE`, Hardwood throws an
`UnsupportedOperationException` reporting that the encoding is not supported
for that type.

This is an encoding/type compatibility failure, not a decompression failure.
