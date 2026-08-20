# Answers — Chapter 8: RLE/Bit Packing and Dictionaries

Return to the [lesson](../chapters/08-rle-and-dictionaries.md).

## 1. Bit width for 0 through 5

Five is binary `101`. It needs three bits:

```text
2 bits represent at most 3
3 bits represent at most 7
```

The required width is **3**.

## 2. RLE run of nine value-5 entries

The RLE header is the count shifted left:

```text
9 << 1 = 18 decimal = 12 hex
```

At bit width 3, the repeated value occupies:

```text
ceil(3 / 8) = 1 byte
```

Value 5 is `05`, so the complete run is:

```text
12 05
```

## 3. Two packed groups at width 3

The header is:

```text
(2 << 1) | 1 = 5 = 05
```

Two groups represent:

```text
2 groups × 8 values/group = 16 values
```

Their payload occupies:

```text
2 groups × 3 bytes/group = 6 bytes
```

Thus `05` is followed by six packed bytes.

## 4. Decode `0A 03 03 E4 E4`

At width 2:

1. `0A` is even. `0A >>> 1 = 5`, so read one value byte `03` and emit five
   copies of 3.
2. The next `03` is odd. `03 >>> 1 = 1`, so decode one group of eight from two
   bytes.
3. Repeated two-bit masking of `E4` gives `0, 1, 2, 3`. The second `E4` gives
   the same four values.

Result:

```text
3,3,3,3,3,0,1,2,3,0,1,2,3
```

## 5. Header for an RLE run of 100

First form the numeric header:

```text
100 << 1 = 200
```

Encode 200 as unsigned LEB128. The low seven bits are 72 (`0x48`); because
more bits remain, set the continuation bit:

```text
0x48 | 0x80 = C8
remaining = 200 >>> 7 = 1
```

The final group is `01`, so the varint is **`C8 01`**. The repeated-value bytes
would follow.

## 6. Width for six dictionary entries

Six entries have indices 0 through 5. Since 5 is binary `101`, the width is
**3 bits**:

```text
ceil(log2(6)) = 3
```

## 7. Definitions plus dense indices

Walk logical positions while advancing the index cursor only at definition 1:

```text
position 0: def 1 -> consume 2
position 1: def 0 -> null, consume nothing
position 2: def 1 -> consume 0
position 3: def 1 -> consume 2
```

The logical index sequence is:

```text
[2, null, 0, 2]
```

There is no null dictionary index.

## 8. Hardwood dictionary branch

`PageDecoder` first consumes one unsigned byte as `bitWidth`. It then creates
an `RleBitPackingHybridDecoder` over the remaining value-region bytes; that
object represents the **dense stream of dictionary indices**, not the final
column values.

The already-parsed typed `Dictionary` receives that decoder. Its
type-specific branch allocates a primitive output array (or `byte[][]`),
decodes only maximum-definition indices, looks each index up in its entry
array, and creates a typed `Page`. Thus the next created data structure
represents **resolved physical values in logical page positions**, with levels
carried alongside it.
