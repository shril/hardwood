# Answers — Chapter 7: PLAIN Values and Nullability

Return to the [lesson](../chapters/07-plain-and-nulls.md).

## 1. Level and dense counts

There are five definition entries because the page represents five logical
positions. Maximum definition 1 occurs at indexes 0, 2, and 4:

```text
count(def == 1) = 3
```

Therefore the page has **five level entries and three dense values**.

## 2. PLAIN `INT32` bytes

PLAIN `INT32` uses four-byte little-endian order:

```text
1   = 0x00000001 -> 01 00 00 00
258 = 0x00000102 -> 02 01 00 00
```

Back-to-back, the value region is:

```text
01 00 00 00 02 01 00 00
```

## 3. PLAIN Boolean byte

The first Boolean occupies bit 0 and the eighth occupies bit 7:

```text
bit:    7 6 5 4 3 2 1 0
value:  1 0 0 0 0 1 0 1
```

That is binary `10000101`, or **`85`** in hexadecimal.

## 4. Present empty `BYTE_ARRAY`

Every present `BYTE_ARRAY` starts with its four-byte little-endian length. An
empty value has length zero and no following content:

```text
00 00 00 00
```

Its definition level must still reach the maximum. The bytes alone are a
zero-length value, not a null marker.

## 5. Dense PLAIN byte length

The null contributes no value bytes:

```text
"cat": 4-byte length + 3 bytes = 7
"":    4-byte length + 0 bytes = 4
"OK":  4-byte length + 2 bytes = 6
total: 7 + 4 + 6 = 17 bytes
```

The definition-level bytes and V1 length prefix are outside this 17-byte value
region.

## 6. Decoder positions

Start at `pos = 0`.

1. `"cat"` consumes 4 length bytes and 3 content bytes: `pos = 7`.
2. The null consumes no value bytes: `pos = 7`.
3. `""` consumes its 4-byte zero length: `pos = 11`.
4. `"OK"` consumes 4 length bytes and 2 content bytes: `pos = 17`.

The unchanged position at the null is the important invariant.

## 7. Logical values from values and definitions

The logical sequence is:

```text
[7, null, 0]
```

At index 1, definition 0 is below maximum 1, so the array's zero is an unused
Java default. At index 2, definition 1 reaches the maximum, so its zero is a
real encoded value. Equal primitive bits can therefore have different logical
meanings depending on validity.

## 8. Why `readInts` counts definitions first

The nullable output array length is the page position count, but only defined
positions have bytes. Counting them lets Hardwood compute the exact readable
byte span:

```text
numBytes = numDefined × 4
```

It can reject truncated input before reading and construct an `IntBuffer` over
only the dense encoded values. The later loop places each consumed integer at
its logical output position while skipping null positions.
