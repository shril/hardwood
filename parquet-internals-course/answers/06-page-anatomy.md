# Answers — Chapter 6: Pages, Headers, Compression, and CRC

Return to the [lesson](../chapters/06-page-anatomy.md).

## 1. Next page offset

The body starts after the 17-byte header:

```text
400 + 17 = 417
```

It occupies 83 stored bytes:

```text
417 + 83 = 500
```

The next page starts at offset **500**. Equivalently:

```text
page start + header size + compressed_page_size
= 400 + 17 + 83
= 500
```

Using `400 + 83` would overlap the current page because the size field excludes
the header.

## 2. V2 size fields

Both fields include the raw level bytes. Only their treatment of the value
region differs:

```text
compressed_page_size
= 4 rep + 6 def + 30 stored values
= 40 bytes

uncompressed_page_size
= 4 rep + 6 def + 90 expanded values
= 100 bytes
```

The decompressor receives 30 bytes and is expected to produce 90, not 40 bytes
and 100.

## 3. Why V1 must be decompressed first

V1 applies the column codec to the body as a single unit. The repetition-level
length, repetition bytes, definition-level length, definition bytes, and value
bytes are all inside that compressed unit. Compression does not preserve
internal byte offsets. The first four stored bytes are codec data, not
necessarily a readable level length, so the reader must first recover the
whole uncompressed body.

## 4. Location of level byte lengths

For a V1 RLE level stream, a **four-byte little-endian length prefix in the
uncompressed body** precedes the hybrid bytes. The prefix itself is part of the
body.

For V2, the lengths are the
`repetition_levels_byte_length` and
`definition_levels_byte_length` **fields in `DataPageHeaderV2`**. The V2 body
contains only the indicated level bytes; it has no four-byte level prefixes.

## 5. CRC coverage

The CRC covers exactly the stored page body:

```text
[body start, body start + compressed_page_size)
```

It excludes the Thrift page header. For V1 this is the compressed whole body.
For V2 it is raw repetition bytes, raw definition bytes, and the stored
possibly-compressed value bytes.

## 6. Three level entries but eight value bytes

The page represents three logical positions: `10`, `null`, and `20`, so it
needs three definition levels: `[1, 0, 1]`.

Only positions whose definition level reaches `maxDefinitionLevel = 1` consume
a value. Two positions qualify. PLAIN `INT32` uses four bytes per value:

```text
2 non-null values × 4 bytes/value = 8 bytes
```

The null is represented by its definition level and has no placeholder bytes
in the on-disk value stream.

## 7. Passing V2 levels to the codec

The codec expects its input to start at the first compressed value byte. If
five raw level bytes are prepended, its framing or first symbols are wrong.
Several outcomes are possible: immediate “invalid compressed data,” a length
mismatch, or meaningless output. Requesting only the value output length does
not repair the input boundary. Correct arithmetic is:

```text
value input offset = rep length + def length = 5
value input length = compressed_page_size - 5
value output length = uncompressed_page_size - 5
```

## 8. Header and body sizing evidence

`PageDecoder` constructs a `ThriftCompactReader`, reads the `PageHeader`, and
uses `headerReader.getBytesRead()` as the serialized header size. That runtime
position is necessary because Compact Protocol headers have variable length.

It then uses `pageHeader.compressedPageSize()` to size the body slice. The two
measurements come from different sources:

- bytes consumed by parsing determine where the body starts;
- the parsed `compressed_page_size` field determines where the body ends.
