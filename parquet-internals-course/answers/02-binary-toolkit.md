# Answers — Chapter 2: Binary Tools for a Java Reader

Return to
[`../chapters/02-binary-toolkit.md`](../chapters/02-binary-toolkit.md).

## 1. One byte in three views

Split `D6` into nibbles:

```text
D = 13 = 1101
6 =  6 = 0110
```

Therefore:

```text
0xD6 = 11010110
```

Its unsigned value is:

```text
13 × 16 + 6 = 214
```

As a Java `byte`, the high bit is set, so the signed value is:

```text
214 - 256 = -42
```

The bits did not change; only their numeric interpretation changed.

## 2. Little-endian fixed integer

For `34 12 00 00`, the first byte contributes the `256^0` place:

```text
0x34 × 256^0 = 52 × 1   =   52
0x12 × 256^1 = 18 × 256 = 4,608
0x00 × 256^2 = 0
0x00 × 256^3 = 0
total                       4,660
```

The result is `0x00001234`, or **4,660**.

## 3. Unsigned varint `AC 02`

For the first byte:

```text
0xAC & 0x7F = 0x2C = 44
0xAC & 0x80 != 0, so continue
```

For the second:

```text
0x02 & 0x7F = 2
0x02 & 0x80 == 0, so stop
```

Seven bits preceded the second payload, so:

```text
44 + (2 << 7)
= 44 + (2 × 128)
= 44 + 256
= 300
```

## 4. ZigZag

ZigZag alternates around zero:

```text
-3 -> 5
 3 -> 6
```

For encoded value 7:

```text
(7 >>> 1) ^ -(7 & 1)
= 3 ^ -1
= -4
```

Equivalently, every odd ZigZag number `2n - 1` represents `-n`; `7 = 2×4 -
1`, so it decodes to **-4**.

## 5. Minimum bit widths

The answers are:

| maximum | width | reason |
|---:|---:|---|
| 0 | 0 | only the constant zero is possible |
| 1 | 1 | one bit represents 0–1 |
| 6 | 3 | two bits stop at 3; three represent 0–7 |
| 7 | 3 | three bits represent 0–7 |
| 8 | 4 | three bits stop at 7 |

## 6. Java byte widening

The values are:

```text
wrong = -24
right = 232
```

`0xE8` has the high bit set. Casting it to `byte` gives the signed value -24,
and normal widening preserves that sign. `& 0xFF` keeps only the low eight bits
in the widened `int`, yielding the unsigned interpretation:

```text
0xE8 = 14 × 16 + 8 = 232
```

## 7. Bounds at the end of a buffer

The proposed body occupies:

```text
[40, 40 + 5) = [40, 45)
```

A 44-byte buffer has valid offsets `[0, 44)`. Because 45 is past its exclusive
end, the decoder must reject the field.

The overflow-safe check is:

```text
offset <= size - length
40     <= 44 - 5
40     <= 39     // false
```

## 8. Format rule or Hardwood choice

1. **Format rule:** the file tail stores metadata length as four-byte
   little-endian.
2. **Hardwood choice:** packing the decoded field id and type into one Java
   `int` is an allocation-avoidance detail of `ThriftCompactReader`.
3. **Format rule:** Parquet mandates Thrift Compact for metadata, and that
   protocol represents signed `i32` values by ZigZag followed by unsigned
   varint.
4. **Hardwood choice:** Java exception type, validation point, and message
   wording are implementation behavior. The reader must not silently return
   wrong data, but the Parquet format does not prescribe this Java diagnostic.
