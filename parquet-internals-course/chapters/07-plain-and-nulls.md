# Chapter 7 — PLAIN Values and Nullability

Allow 45–75 minutes. This chapter separates “positions represented by the
page” from “values that occupy bytes.”

## Objectives

After this chapter you should be able to:

1. hand-decode common PLAIN physical values;
2. explain why nulls consume level entries but no value bytes;
3. compute the number of dense values from definition levels;
4. distinguish an empty byte array from a null byte array;
5. trace PLAIN decoding into Hardwood primitive arrays and validity; and
6. diagnose a value-stream alignment error.

## Prerequisite recap

Chapter 2 introduced little-endian integers and bit order. Chapter 5 separated
physical type from logical interpretation. Chapter 6 located the value region
after levels and decompression.

For this chapter, assume the reader already has:

- the page's `num_values`;
- a definition level for every represented position, when the column is
  nullable;
- the leaf's `maxDefinitionLevel`; and
- the uncompressed value-region bytes.

## Central mental model: two synchronized streams with different lengths

Think of a nullable leaf as two cursors:

```text
position cursor:  0       1       2       3
definition:       max     low     max     max
logical result:   value   null    value   value
                         \_____________________/
value cursor:      0               1       2
```

The level stream is position-dense: it has one entry for every page position.
The value stream is value-dense: it has entries only where the definition level
reaches the leaf's maximum.

**Parquet format rule:** nulls are represented by definition levels. They do
not insert PLAIN zeroes, empty strings, or any other placeholder into the
encoded value stream.

For a flat optional leaf:

```text
non-null count = count(definitionLevel[i] == maxDefinitionLevel)
```

For nested data, lower definitions can also stand for null or empty ancestors.
The same dense-value rule still holds: consume a leaf value only at maximum
definition.

## PLAIN by physical type

PLAIN is type-specific, despite its name.

| Physical type | PLAIN representation |
|---|---|
| `INT32` | 4 bytes, little-endian |
| `INT64` | 8 bytes, little-endian |
| `FLOAT` | 4-byte IEEE 754 bits, little-endian |
| `DOUBLE` | 8-byte IEEE 754 bits, little-endian |
| `BOOLEAN` | one bit per dense value, least-significant bit first |
| `BYTE_ARRAY` | 4-byte little-endian length, then that many bytes |
| `FIXED_LEN_BYTE_ARRAY` | exactly the schema's `type_length` bytes |
| `INT96` | exactly 12 bytes |

**Parquet format rule:** the bytes encode physical values. A PLAIN
`BYTE_ARRAY` annotated as UTF-8 still has a four-byte length and raw bytes;
string decoding is a later logical conversion.

Boolean bits are packed across the dense Boolean sequence. The last byte may
have unused high bits. Null Boolean positions do not consume bits.

## Hand-worked byte/value example

Take a flat optional `BYTE_ARRAY` column:

```text
["cat", null, "", "OK"]
```

Its maximum definition level is 1:

```text
position:  0  1  2  3
def:       1  0  1  1
```

There are four level entries but three dense values. A one-bit bit-packed
hybrid group for the definitions is:

```text
header = 03                 # one group of eight
bits   = 0D                 # 1,0,1,1,0,0,0,0, LSB first
```

Now encode only the three present byte arrays:

```text
"cat" -> 03 00 00 00 63 61 74
""    -> 00 00 00 00
"OK"  -> 02 00 00 00 4F 4B
```

The complete dense PLAIN value region is:

```text
03 00 00 00 63 61 74  00 00 00 00  02 00 00 00 4F 4B
```

Its length is:

```text
(4 + 3) + (4 + 0) + (4 + 2) = 17 bytes
```

For a V1 page, adding the definition stream gives:

```text
02 00 00 00  03 0D  03 00 00 00 63 61 74  00 00 00 00  02 00 00 00 4F 4B
^^^^^^^^^^^   ^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
def byte len  defs   dense PLAIN values
```

The uncompressed body is `4 + 2 + 17 = 23` bytes.

Decode with two cursors:

1. position 0 has `def = 1`: consume `"cat"`;
2. position 1 has `def = 0`: mark null and consume no bytes;
3. position 2 has `def = 1`: consume the zero-length value `""`;
4. position 3 has `def = 1`: consume `"OK"`.

The zero-length value is not null. Its definition reaches the maximum, and its
four-byte length field is zero.

As a second quick example, dense PLAIN Booleans
`[true, false, true, true]` occupy one byte:

```text
bit positions: 3 2 1 0
values:        1 1 0 1
byte:          00001101 = 0D
```

## Format rules and Hardwood choices

**Parquet format rule**

- PLAIN representation is selected by physical type.
- Fixed-width numeric payloads are little-endian.
- PLAIN Boolean values are packed one bit each, least-significant bit first.
- PLAIN `BYTE_ARRAY` includes a four-byte length for every non-null value.
- Only maximum-definition positions consume value entries.

**Hardwood choice**

- Hardwood decodes into primitive arrays (`int[]`, `long[]`, `float[]`,
  `double[]`, and `boolean[]`) to avoid boxing.
- Output arrays have one slot per page position. Slots for null positions keep
  Java's default value, but validity—not the default value—determines nullness.
- Byte-array-backed physical types use `byte[][]` at the decoded-page boundary.
- An all-present definition stream can be represented internally by a null
  level array. This is an optimization signal, not a Parquet on-disk rule.

For `[10, null, 20]`, Hardwood may hold:

```text
values:   [10, 0, 20]
validity: [present, null, present]
```

The middle zero is not data and must never be used to infer that the row was
non-null.

## Current Hardwood code tour

1. [`PlainDecoder`](../../core/src/main/java/dev/hardwood/internal/encoding/PlainDecoder.java)
   has one primitive method per type family. In `readInts`, compare the
   all-present bulk path with the loop that advances the `IntBuffer` only at
   maximum definition.
2. In the same class, inspect `readBoolean`, `readByteArray`, and
   `readFixedLenByteArray`. Note the Boolean bit cursor and little-endian
   `BYTE_ARRAY` length.
3. [`PageDecoder.decodeTypedValues`](../../core/src/main/java/dev/hardwood/internal/reader/PageDecoder.java)
   allocates arrays of `numValues`, chooses a `PlainDecoder`, and carries the
   levels into the resulting typed `Page`.
4. [`Page`](../../core/src/main/java/dev/hardwood/internal/reader/Page.java)
   defines `isNull` from definition levels. Its `size()` is authoritative
   because pooled level arrays may be physically longer than the current page.
5. [`ColumnReader`](../../core/src/main/java/dev/hardwood/reader/ColumnReader.java)
   exposes typed arrays separately from `getLeafValidity()`.
6. [`ParquetReaderTest`](../../core/src/test/java/dev/hardwood/ParquetReaderTest.java)
   checks the fixture's `"alice", null, "charlie"` sequence.

## Guided lab

### 1. Inspect a known PLAIN fixture

The fixture is generated in
[`tools/simple-datagen.py`](../../tools/simple-datagen.py) with dictionaries
and compression disabled:

[`plain_uncompressed_with_nulls.parquet`](../../core/src/test/resources/plain_uncompressed_with_nulls.parquet)

With a built CLI:

```shell
hardwood schema -f core/src/test/resources/plain_uncompressed_with_nulls.parquet
hardwood inspect pages -f core/src/test/resources/plain_uncompressed_with_nulls.parquet
hardwood print -f core/src/test/resources/plain_uncompressed_with_nulls.parquet
```

Record:

- the physical type and repetition of `name`;
- the page's value encoding; and
- the visible difference between the null row and the two strings.

### 2. Run the focused test

```shell
timeout 180s ./mvnw -pl core \
  -Dtest=ParquetReaderTest#testReadPlainParquetWithNulls test
```

In the test, find the separate accesses to `getBinaries()` and
`getLeafValidity()`. Explain why asserting only `nameValues[1]` would not be a
complete null check for every physical type.

### 3. Trace cursor movement

In
[`PlainDecoder.readByteArrays`](../../core/src/main/java/dev/hardwood/internal/encoding/PlainDecoder.java),
trace the hand-worked definitions `[1, 0, 1, 1]`. Write down `pos` after
decoding `"cat"`, after the null, after `""`, and after `"OK"`, starting at
zero. Use the 17-byte value region, not the level bytes.

Expected checkpoints are available in the answer key, but calculate them
first.

## Common misconceptions

- **“`num_values` is the number of non-null values.”** It is the count of page
  positions/level entries; the dense value count can be smaller.
- **“Null integers are encoded as zero.”** Nulls have no integer bytes. Zero in
  a Java output slot is only an unused default.
- **“A zero-length `BYTE_ARRAY` is null.”** Its maximum definition level makes
  it present; length zero makes it empty.
- **“PLAIN means no structure.”** Variable-length values still carry lengths,
  and Booleans still share bytes.
- **“Logical strings change PLAIN layout.”** UTF-8 interpretation happens
  after physical `BYTE_ARRAY` decoding.
- **“The definition array length is always the current page size.”** Hardwood
  may reuse a larger scratch array; iterate `Page.size()`.

## Recap and debugging checklist

- [ ] Identify the physical type before decoding PLAIN.
- [ ] Iterate exactly the page's logical position count.
- [ ] Consume a value only when `definition == maxDefinition`.
- [ ] Keep a separate dense-value cursor.
- [ ] Use validity to identify nulls; never inspect placeholder values.
- [ ] Treat empty and null byte arrays as distinct.
- [ ] Check little-endian order and fixed widths.
- [ ] For Booleans, advance one bit per non-null value, LSB first.

## Quiz

1. An optional flat page has definitions `[1, 0, 1, 0, 1]`. How many level
   entries and how many dense values does it have?
2. Encode PLAIN `INT32` values `1` and `258` as hexadecimal bytes.
3. Encode dense PLAIN Booleans
   `[true, false, true, false, false, false, false, true]` into one byte.
4. What bytes represent the present empty `BYTE_ARRAY` value?
5. For `["cat", null, "", "OK"]`, calculate the dense PLAIN byte length.
6. Starting at value-region offset zero in that example, what is the decoder
   position after each of the four logical positions?
7. A decoded `int[]` is `[7, 0, 0]` with definitions `[1, 0, 1]`. What are the
   logical values, and why?
8. In Hardwood's `PlainDecoder.readInts`, why does the nullable branch first
   count defined positions before creating its `IntBuffer`?

Compare your work with the
[answer key](../answers/07-plain-and-nulls.md).
