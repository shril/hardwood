# Answers — Chapter 13: Decoding a Page

[Return to the lesson](../chapters/13-page-decoding.md)

## 1. Operations before the V1/V2 branch

`decodePage`:

1. starts the decode event;
2. parses `PageHeader` from the complete page buffer;
3. obtains `headerSize` from the Thrift reader;
4. reads `compressedPageSize` and slices exactly that body region;
5. validates the optional CRC over the stored body.

Only then does page type choose `DATA_PAGE` or `DATA_PAGE_V2`. Decompression
cannot safely happen earlier because the two types have different compressed
regions.

## 2. Why definition levels come first

An optional column does not encode a physical value for a null level
position. The value decoder needs definition levels to decide whether to
consume bytes and where to place each decoded primitive in the logical output
array.

Without levels, it would consume the next value for a null position, shift
every later value left, and likely either leave unread bytes or overrun the
input.

## 3. Sparse slots for optional INT32

The result is:

```text
values = [4, 0, 5, 0, 6]
size = 5
readable/present indices = 0, 2, 4
```

Indices 1 and 3 contain Java's default zero only because no encoded integer
was consumed for them. Their definition levels mark them absent, so those
zeros are not semantic values.

## 4. V2 uncompressed value size

The expected value-region size is:

```text
136 - 10 - 6 = 120 bytes
```

The decompressor receives the 80 stored value bytes and is asked to produce
120 bytes. The 16 level bytes are already uncompressed and are not codec
input.

## 5. Why level arrays can be longer

Each reorder-buffer slot owns a `LevelScratch`. Its arrays grow to the largest
page decoded through that slot and are reused for later smaller pages. A page
record can therefore point at an array with a stale tail beyond its own
`size()`.

Iterating to array length can assemble old levels as new rows, consume default
or stale value slots, break row counts, and misalign columns. `Page.size()` is
the only valid logical bound.

## 6. Dictionary preconditions

Before dictionary index decoding:

1. a parsed `Dictionary` must be present;
2. the unsigned index bit width byte must be in `[0, 32]`.

A missing dictionary causes `IOException("Dictionary page not found ...")`.
An excessive width causes an `IOException` identifying the invalid width and
column. Hardwood does not guess a dictionary or truncate the width.

Earlier in the page path, optional CRC validation also protects the stored
body, but the two checks above are the dictionary-dispatch preconditions.

## 7. Binary page representation

`INT32` values all have one fixed primitive width, so `int[]` is sufficient.
`BYTE_ARRAY` values have independent lengths; Hardwood's page representation
uses `byte[][]`, one byte array reference per logical position. It is
array-based but not one flat primitive array.

A dictionary-backed `ByteArrayPage` can also retain the shared
`ByteArrayDictionary` and an `int[]` of per-position dictionary indices.
Downstream binary/string assembly can reuse or intern dictionary entries
instead of treating every repeated value as unrelated bytes.

## 8. Best regression boundary for wrong V2 codec input

Start at a focused `PageDecoder`-level test. The codec may be perfectly
correct when given its input; the bug is that V2 page parsing selected the
wrong input range. A codec-only test cannot observe that boundary.

Construct or use a fixture/page with nonempty uncompressed levels and
compressed values, then assert decoded levels and values. Add a public
end-to-end regression if the report came from public reading or if logical
assembly needs protection, but keep the first failing invariant at
`PageDecoder`.
