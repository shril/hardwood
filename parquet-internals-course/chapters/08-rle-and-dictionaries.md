# Chapter 8 — RLE/Bit Packing and Dictionaries

Allow 45–75 minutes. This chapter turns repeated small integers into run
packets, then uses those integers as dictionary addresses.

## Objectives

After this chapter you should be able to:

1. compute bit width for levels and dictionary indices;
2. distinguish RLE and bit-packed hybrid run headers;
3. decode unsigned-varint run headers, including multi-byte headers;
4. unpack groups of eight values in least-significant-bit order;
5. separate a dictionary page from data-page index streams; and
6. trace dictionary decoding through current Hardwood classes.

## Prerequisite recap

Chapter 2 introduced bit width and unsigned varints. Chapter 6 located level
regions, and Chapter 7 established that level streams have one entry per
position while data-page value streams are dense.

For a non-negative maximum `m`, the width needed to represent values from zero
through `m` is:

```text
bitWidth(0) = 0
bitWidth(m) = floor(log2(m)) + 1, when m > 0
```

For levels, `m` is the maximum level. For dictionary indices, `m` is the
largest dictionary index.

## Central mental model: a stream of self-describing runs

The hybrid encoding alternates between two packet types. Every packet begins
with an unsigned varint:

```text
header low bit 0 -> RLE run
header low bit 1 -> bit-packed run
```

### RLE run

```text
header = repeatCount << 1
payload = repeated value in ceil(bitWidth / 8) little-endian bytes
```

Decode with:

```text
repeatCount = header >>> 1
```

At bit width 2, five copies of value 3 are:

```text
header: 5 << 1 = 10 = 0A
value:  03
bytes:  0A 03
```

Run headers are varints, not always one byte. One hundred copies have header
`100 << 1 = 200`. Unsigned LEB128 splits 200 into seven-bit groups:

```text
200 decimal = 0xC8
varint       = C8 01
```

At bit width 2, one hundred copies of value 3 are therefore:

```text
C8 01 03
```

### Bit-packed run

Bit-packed runs contain whole groups of eight values:

```text
header = (groupCount << 1) | 1
value count = groupCount × 8
payload bytes = groupCount × bitWidth
```

Why `groupCount × bitWidth` bytes? Eight values times `bitWidth` bits is
`8 × bitWidth` bits, exactly `bitWidth` bytes.

Values are packed consecutively with each value's least-significant bit first.
A final group may contain padding values; the enclosing page count tells the
consumer when to stop.

**Parquet format rule:** the low header bit selects the run type. The remaining
bits have different units: individual values for RLE, groups of eight for
bit-packed runs.

## Hand-worked hybrid example

Use bit width 2 and this sequence:

```text
3,3,3,3,3, 0,1,2,3,0,1,2,3
```

Encode the first five values as RLE:

```text
0A 03
```

Encode the next eight as one packed group. Its header is:

```text
(1 << 1) | 1 = 3 = 03
```

Four two-bit values fit in each byte:

```text
first four:  0,1,2,3
bits:        00 | 01<<2 | 10<<4 | 11<<6
decimal:     0 + 4 + 32 + 192 = 228 = E4

next four:   0,1,2,3 -> E4
```

The complete hybrid bytes are:

```text
0A 03  03 E4 E4
^^^^^  ^^^^^^^^
RLE    packed group
```

Decoding:

1. `0A` is even, so repeat count is `10 >>> 1 = 5`; read value `03`.
2. `03` is odd, so group count is `3 >>> 1 = 1`; read
   `1 × 2 = 2` payload bytes.
3. Mask two low bits repeatedly from `E4 E4` to recover
   `0,1,2,3,0,1,2,3`.

### Where outer lengths live

The hybrid stream itself is a sequence of runs. Its container tells the reader
where it ends:

- V1 RLE level region: four-byte little-endian byte length before the stream;
- V2 level region: byte length in `DataPageHeaderV2`;
- dictionary index region: a one-byte bit width, followed by hybrid runs to the
  end of the value region—no V1-style four-byte hybrid length.

Do not move a length convention from one container to another.

## Dictionary encoding: values once, indices many times

A dictionary-encoded column chunk has two connected pieces:

```text
dictionary page:       entry 0, entry 1, entry 2, ...
data page value area:  bitWidth byte + hybrid-encoded indices
```

**Parquet format rule:** dictionary entries are stored in a dictionary page
using PLAIN representation. Data pages marked `RLE_DICTIONARY` (or deprecated
`PLAIN_DICTIONARY`) carry indices. The dictionary page precedes data pages that
use it and belongs to that column chunk.

Suppose the dictionary is:

```text
0 -> "red"
1 -> "green"
2 -> "blue"
```

The largest index is 2, so the required width is 2 bits. Consider logical
values:

```text
["red", null, "green", "red", "blue", "green"]
```

Definitions are `[1, 0, 1, 1, 1, 1]`. The dense index sequence is:

```text
0,1,0,2,1
```

Pad one packed group to eight:

```text
0,1,0,2,1,0,0,0
```

Packing the first four gives:

```text
0 | 1<<2 | 0<<4 | 2<<6 = 132 = 84
```

The next four give `01`. The data-page value bytes are:

```text
02  03 84 01
^^  ^^^^^^^^
BW  one packed group
```

The first `02` is not a hybrid header; it is the dictionary-index bit width.
Only five decoded indices are consumed because only five definitions reach the
maximum.

The PLAIN dictionary body itself is:

```text
03 00 00 00 72 65 64
05 00 00 00 67 72 65 65 6E
04 00 00 00 62 6C 75 65
```

Its length is `(4+3) + (4+5) + (4+4) = 24` bytes.

For a one-entry dictionary, the largest index is zero and bit width is zero.
Every dense value implicitly selects entry zero; an implementation must handle
that case without trying to read index payload bits.

## Format rules and Hardwood choices

**Parquet format rule**

- Hybrid run headers are unsigned varints.
- Even headers are RLE counts; odd headers count groups of eight.
- RLE values occupy `ceil(bitWidth / 8)` little-endian bytes.
- Bit-packed payloads consume `groupCount × bitWidth` bytes.
- Dictionary data-page value areas begin with a one-byte index bit width.
- Null positions consume levels but no dictionary index.

**Hardwood choice**

- Hardwood accepts hybrid widths from 0 through 32 and rejects others.
- Its decoder fills primitive arrays directly and uses a reusable temporary
  primitive index buffer for dictionary lookup.
- Dictionary objects are typed (`IntDictionary`, `LongDictionary`, and so on).
- Byte-array dictionaries retain entry indices so higher layers can reuse one
  decoded `String` per dictionary entry.

## Current Hardwood code tour

1. [`RleBitPackingHybridDecoder`](../../core/src/main/java/dev/hardwood/internal/encoding/RleBitPackingHybridDecoder.java)
   reads the unsigned varint in `readNextRun`. Compare the even and odd
   branches.
2. In `readRleValue`, verify the byte count and little-endian shifts. In
   `decodeBitPacked`, find the two-bit-style mask-and-shift loop.
3. [`DictionaryParser`](../../core/src/main/java/dev/hardwood/internal/reader/DictionaryParser.java)
   parses the dictionary page header, validates CRC, decompresses the body, and
   delegates typed PLAIN parsing.
4. [`Dictionary`](../../core/src/main/java/dev/hardwood/internal/reader/Dictionary.java)
   stores typed entry arrays and applies decoded indices without boxing.
5. [`PageDecoder`](../../core/src/main/java/dev/hardwood/internal/reader/PageDecoder.java)
   handles `RLE_DICTIONARY` and `PLAIN_DICTIONARY`: it reads the first value
   byte as width, constructs the hybrid decoder over the remaining bytes, and
   asks the dictionary to build a typed page.
6. [`NestedDictBatchBoundaryTest`](../../core/src/test/java/dev/hardwood/NestedDictBatchBoundaryTest.java)
   shows dictionary decoding surviving nulls and batch boundaries in the full
   reader path.

## Guided lab

### 1. Inspect a dictionary page and entries

Use the checked-in fixture:

[`dictionary_uncompressed.parquet`](../../core/src/test/resources/dictionary_uncompressed.parquet)

Its `category` values are `A, B, A, C, B`.

```shell
hardwood inspect pages -f core/src/test/resources/dictionary_uncompressed.parquet \
  --column category
hardwood inspect dictionary -f core/src/test/resources/dictionary_uncompressed.parquet \
  --column category --limit 0
```

Confirm that:

- a dictionary page appears before the data page;
- dictionary entries are unique while printed rows can repeat them; and
- the data page reports a dictionary encoding.

### 2. Run focused dictionary tests

```shell
timeout 180s ./mvnw -pl core \
  -Dtest=CrcValidationTest#testReadDictionaryFileWithCrc test

timeout 180s ./mvnw -pl core \
  -Dtest=NestedDictBatchBoundaryTest test
```

The first follows a five-value dictionary fixture through CRC and lookup. The
second uses a dictionary-encoded nested string with parent and leaf nulls.

### 3. Trace the hand bytes in code

Starting at
[`RleBitPackingHybridDecoder.readNextRun`](../../core/src/main/java/dev/hardwood/internal/encoding/RleBitPackingHybridDecoder.java),
trace `0A 03 03 E4 E4` with bit width 2. At each run write:

- `header`;
- `isRleRun`;
- `remainingInRun`; and
- the new byte position.

Then trace dictionary bytes `02 03 84 01`: account for the leading width before
constructing the hybrid decoder.

## Common misconceptions

- **“An odd header is an odd number of values.”** It counts groups of eight
  after shifting right.
- **“A run header is one byte.”** It is an unsigned varint.
- **“Bit-packed payloads can contain any count.”** Hybrid packed runs store
  whole groups of eight; the consumer count ignores final padding.
- **“The RLE value uses `bitWidth` bytes.”** It uses
  `ceil(bitWidth / 8)` bytes.
- **“Dictionary data pages contain dictionary values.”** They contain indices;
  the dictionary page contains entries.
- **“Nulls need a special dictionary index.”** They consume no index.
- **“The first dictionary value byte is a run header.”** It is the one-byte bit
  width.

## Recap and debugging checklist

- [ ] Compute bit width from the maximum representable value.
- [ ] Parse every run header as an unsigned varint.
- [ ] Test the low bit before interpreting the shifted count.
- [ ] Multiply packed group count by eight for values.
- [ ] Multiply group count by bit width for payload bytes.
- [ ] Stop at the enclosing expected value count, ignoring packed padding.
- [ ] For dictionary data, consume the one-byte width first.
- [ ] Decode indices only for maximum-definition positions.
- [ ] Ensure the dictionary belongs to the same column chunk.

## Quiz

1. What bit width is required for values from 0 through 5?
2. At bit width 3, encode an RLE run of nine copies of value 5.
3. At bit width 3, a packed header says there are two groups. What is the
   header byte, how many values are represented, and how many payload bytes
   follow?
4. Decode `0A 03 03 E4 E4` at bit width 2.
5. What varint header bytes encode an RLE run of 100 values?
6. A dictionary has six entries. What is its index bit width?
7. Definitions are `[1, 0, 1, 1]` and dense dictionary indices are
   `[2, 0, 2]`. Which logical index sequence results?
8. Trace Hardwood's dictionary branch from the first value-region byte to a
   typed output array. What does each of the first two objects created
   represent?

Compare your work with the
[answer key](../answers/08-rle-and-dictionaries.md).
