# Chapter 2 — Binary Tools for a Java Reader

Chapter 1 treated a column as a sequence of values. A file contains only bytes,
so a reader needs rules that turn bytes into lengths, identifiers, integers,
bits, and eventually values.

This chapter introduces the small binary toolkit used by the rest of the
course. The goal is not to become a human disassembler. It is to make every
cursor movement and integer calculation explicit enough that malformed input
cannot silently shift the interpretation of all following bytes.

## Learning objectives

After this chapter, you should be able to:

1. read hexadecimal byte notation and translate between a byte, bits, and an
   unsigned value;
2. explain offsets, lengths, and a decoder cursor;
3. decode a fixed-width little-endian integer;
4. decode a small unsigned varint and explain its continuation bit;
5. map small signed integers through ZigZag encoding;
6. compute the minimum bit width for a known maximum;
7. avoid Java `byte` sign-extension errors; and
8. recognize these operations in Hardwood's binary readers.

## Prerequisite recap

From Chapter 1:

- Parquet groups values into leaf-column streams.
- A reader requests selected ranges through an input boundary.
- Correct decoding requires both values and enough structure to preserve their
  coordinates.

No binary-format knowledge is assumed. We start with a single byte.

## Central mental model: bytes plus a contract plus a cursor

A byte sequence has no inherent meaning:

```text
E8 03 00 00
```

It becomes meaningful only with a **contract**:

```text
contract: four-byte, unsigned, little-endian integer
meaning:  1,000
```

and a **cursor**:

```text
buffer:  [E8 03 00 00] [next field ...]
cursor:   ^

read four bytes

buffer:  [E8 03 00 00] [next field ...]
cursor:               ^
```

Keep this equation:

```text
meaning = bytes + representation rule + position
```

A wrong rule produces a wrong value. A wrong position is worse: it can make
every later field wrong. Robust readers therefore:

1. know how many bytes or termination markers a representation uses;
2. check that those bytes remain;
3. decode without losing unsigned bits; and
4. advance the cursor exactly once.

## Bytes, bits, and hexadecimal

A **bit** is `0` or `1`. A **byte** contains eight bits and therefore has 256
possible patterns.

Binary notation is verbose:

```text
1110 1000
```

Hexadecimal, base 16, gives one digit to every four bits:

| bits | hex | decimal |
|---|---:|---:|
| `0000` | `0` | 0 |
| `0001` | `1` | 1 |
| `0010` | `2` | 2 |
| `0011` | `3` | 3 |
| `0100` | `4` | 4 |
| `0101` | `5` | 5 |
| `0110` | `6` | 6 |
| `0111` | `7` | 7 |
| `1000` | `8` | 8 |
| `1001` | `9` | 9 |
| `1010` | `A` | 10 |
| `1011` | `B` | 11 |
| `1100` | `C` | 12 |
| `1101` | `D` | 13 |
| `1110` | `E` | 14 |
| `1111` | `F` | 15 |

Thus:

```text
E8 = 1110 1000 = 14 × 16 + 8 = 232
```

`0xE8` is Java's conventional notation for hexadecimal `E8`. Two hex digits
describe one byte. Four bytes are often displayed with spaces:

```text
E8 03 00 00
```

The spaces are for humans; they are not stored.

### Nibbles and masks

Half a byte, four bits, is a **nibble**. Some formats pack two small values into
one byte. Given `0xA5`:

```text
high nibble = A = 10
low nibble  = 5 = 5
```

Java extracts them with masks and shifts:

```java
int unsignedByte = 0xA5;
int high = (unsignedByte >>> 4) & 0x0F;
int low = unsignedByte & 0x0F;
```

`0x0F` has its low four bits set, so `& 0x0F` discards the high nibble.
`>>>` shifts zeros in from the left.

## Java's signed `byte`

Java's `byte` ranges from -128 to 127. Files do not change their bit patterns
just because Java's smallest integer is signed. Reading the byte pattern
`1110 1000` produces a Java `byte` whose numeric value is -24.

Widen it safely before arithmetic:

```java
byte raw = (byte) 0xE8;
int unsigned = raw & 0xFF;  // 232
```

Without the mask, assigning `raw` to an `int` sign-extends it:

```text
byte bits:       11101000
wrong widened:   11111111 11111111 11111111 11101000 = -24
masked result:   00000000 00000000 00000000 11101000 = 232
```

Use the signed value only when the format says the byte itself is signed.

## Offsets, lengths, and bounds

An **offset** is a zero-based position. A **length** is a count. The half-open
range `[offset, offset + length)` includes its start and excludes its end.

For a 100-byte file:

```text
readRange(20, 4) -> offsets 20, 21, 22, 23
```

Offset 24 is the first byte after the range.

Checking `offset + length <= fileSize` directly can overflow for large
integers. A safer shape, after rejecting negative values, is:

```text
offset <= fileSize - length
```

This exact concern appears in Hardwood's local-file input.

## Fixed-width integers and endianness

A fixed-width 32-bit integer always occupies four bytes. **Endianness** defines
which byte comes first.

For the mathematical value 1,000:

```text
decimal: 1,000
hex:     0x000003E8

big-endian bytes:    00 00 03 E8
little-endian bytes: E8 03 00 00
```

“Little-endian” means the least significant byte—the `E8` contributing the
smallest place values—is stored first.

Decode `E8 03 00 00` by place:

```text
E8 × 256^0 = 232 × 1   = 232
03 × 256^1 =   3 × 256 = 768
00 × 256^2 = 0
00 × 256^3 = 0
total                     1,000
```

In Java:

```java
import java.nio.ByteBuffer;
import java.nio.ByteOrder;

byte[] bytes = {(byte) 0xE8, 0x03, 0x00, 0x00};
int value = ByteBuffer.wrap(bytes)
        .order(ByteOrder.LITTLE_ENDIAN)
        .getInt();
```

`ByteBuffer` defaults to big-endian. A Parquet decoder must set the order at
the point where a rule requires little-endian data. Never infer the order from
the host CPU.

> **Parquet format rule:** PLAIN `INT32` and `INT64` values are little-endian,
> as are the four bytes storing the file-metadata length at the tail. Many
> other Parquet structures use different rules. “Every Parquet integer is a
> fixed-width little-endian integer” is false.

## Unsigned varints

A **variable-length integer**, or varint, gives small values fewer bytes.
Parquet-related encodings use unsigned LEB128: each byte contributes seven data
bits. The high bit is a continuation flag:

```text
high bit 0 -> this is the final byte
high bit 1 -> another byte follows
```

The lowest seven data bits come first. Values 0 through 127 fit in one byte.

### Worked varint: 300

Split 300 into base-128 digits:

```text
300 = 44 + (2 × 128)
```

The first seven-bit payload is 44 (`0x2C`). Another digit follows, so set its
continuation bit:

```text
0x2C | 0x80 = 0xAC
```

The second payload is 2 and is final:

```text
encoded bytes: AC 02
```

Decode:

```text
AC: payload AC & 7F = 2C = 44; continuation is set
02: payload 02      = 2;      continuation is clear

44 + (2 << 7) = 44 + 256 = 300
```

`<< 7` means multiplication by 128. A decoder advances in seven-bit steps and
must reject an unterminated or overlong varint rather than wrapping a Java
shift.

Varints are not simply “little-endian fixed integers with zeros removed.” They
reserve one bit per byte for continuation and use seven-bit groups.

## ZigZag for signed values

An unsigned varint represents small non-negative numbers compactly. Directly
viewing a negative two's-complement integer as unsigned would make small
negative values enormous. **ZigZag encoding** maps signed numbers around zero:

| signed | ZigZag unsigned |
|---:|---:|
| 0 | 0 |
| -1 | 1 |
| 1 | 2 |
| -2 | 3 |
| 2 | 4 |
| -3 | 5 |

The formulas for a Java `long` are:

```java
long encoded = (value << 1) ^ (value >> 63);
long decoded = (encoded >>> 1) ^ -(encoded & 1);
```

ZigZag is a mapping, not a complete byte representation. Thrift Compact first
ZigZag-maps signed `i16`, `i32`, and `i64` values, then writes the result as an
unsigned varint.

For `-3`:

```text
ZigZag(-3) = 5
varint(5)  = 05
```

For `150`:

```text
ZigZag(150) = 300
varint(300) = AC 02
```

Do not ZigZag lengths and counts whose wire contract says they are unsigned.

## Bit width and bit packing

If values are known to lie between zero and a maximum, they may need fewer than
32 bits each. The **bit width** is the smallest number of bits that can
represent every value in the range.

Examples:

| maximum | representable range needed | bit width |
|---:|---|---:|
| 0 | only 0 | 0 |
| 1 | 0–1 | 1 |
| 2 | 0–2 | 2 |
| 3 | 0–3 | 2 |
| 4 | 0–4 | 3 |
| 7 | 0–7 | 3 |
| 8 | 0–8 | 4 |

For positive `max`:

```text
bitWidth = floor(log2(max)) + 1
```

Hardwood computes this without floating point:

```java
int width = max == 0
        ? 0
        : 32 - Integer.numberOfLeadingZeros(max);
```

**Bit packing** places values back-to-back at that width. Eight one-bit values
fit in one byte. Parquet's current RLE/bit-packed hybrid uses least-significant
bits first within the byte stream. For width 1, byte `0x4D` is:

```text
hex byte: 4D
bits as normally printed, most-significant first: 01001101
values read least-significant bit first:          1,0,1,1,0,0,1,0
```

You do not need the full hybrid encoding yet; Chapter 8 develops it. Here, be
able to compute the width and recognize that bit order is part of the contract.

## One file, several integer representations

Avoid a dangerous shortcut: there is no single “Parquet integer encoding.”

| Context | Representation |
|---|---|
| tail metadata length | fixed 4-byte little-endian |
| PLAIN `INT32` value | fixed 4-byte little-endian |
| Thrift Compact signed `i32` | ZigZag, then unsigned varint |
| Thrift binary length | unsigned varint |
| RLE/bit-packed run header | unsigned varint |
| packed levels or dictionary indices | fixed bit width, LSB-first |

The caller must know the context before choosing a decoder.

## Format rules and implementation choices

> **Parquet format rule:** The file metadata and page headers are serialized
> from Parquet's Thrift definitions using Thrift Compact Protocol.

> **Parquet format rule:** PLAIN native integer and floating-point
> representations use little-endian byte order; PLAIN `BYTE_ARRAY` uses a
> four-byte little-endian length followed by that many bytes.

> **Parquet format rule:** RLE/bit-packed hybrid run headers are unsigned
> varints, and the bit width is known from context.

> **Hardwood implementation choice:** Hardwood uses Java `ByteBuffer`, primitive
> arrays, masks, and shifts rather than a generated Thrift runtime for these
> paths.

> **Hardwood implementation choice:** `ThriftCompactReader` returns a packed
> `int` for a field header instead of allocating a header object.

> **Hardwood implementation choice:** Input validation turns truncation,
> impossible lengths, and overlong varints into controlled exceptions instead
> of allowing unchecked allocation or cursor desynchronization.

## Current Hardwood code tour

1. [`MappedInputFile.readRange`](../../core/src/main/java/dev/hardwood/internal/reader/MappedInputFile.java)
   checks offset and length without overflowing `offset + length`.
2. [`PlainDecoder`](../../core/src/main/java/dev/hardwood/internal/encoding/PlainDecoder.java)
   sets `ByteOrder.LITTLE_ENDIAN` for fixed-width values, masks bytes while
   reading booleans, and bounds-checks variable byte arrays.
3. [`LevelEncoder.bitWidth`](../../core/src/main/java/dev/hardwood/internal/encoding/LevelEncoder.java)
   shows the integer-only width calculation.
4. [`ThriftCompactReader`](../../core/src/main/java/dev/hardwood/internal/thrift/ThriftCompactReader.java)
   contains `readVarint`, `readZigzag`, fixed-width `readDouble`, binary-length
   checks, and cursor-preserving skip logic.
5. [`RleBitPackingHybridDecoder`](../../core/src/main/java/dev/hardwood/internal/encoding/RleBitPackingHybridDecoder.java)
   masks each Java byte, decodes an unsigned run header, and extracts packed
   values from low bits.
6. [`ThriftCompactReaderTest`](../../core/src/test/java/dev/hardwood/internal/thrift/ThriftCompactReaderTest.java)
   includes adversarial cases for oversized collections, overlong varints,
   field-id overflow, and nested skipping.

Read code in terms of the mental model:

```text
Which contract? -> Are enough bytes present? -> Decode -> Advance how far?
```

## Guided lab: inspect and decode boundary bytes

Run from the repository root.

### 1. Display bytes in hexadecimal

```shell
FILE=core/src/test/resources/plain_uncompressed.parquet
SIZE=$(stat -c %s "$FILE")
od -An -tx1 -N16 "$FILE"
od -An -tx1 -j $((SIZE-16)) -N16 "$FILE"
```

`od -tx1` displays one-byte hexadecimal units. For the checked-in fixture, the
first four bytes and final four bytes are:

```text
50 41 52 31
```

Use an ASCII table or `printf` to verify:

```shell
printf '\x50\x41\x52\x31\n'
```

The result is `PAR1`. Chapter 3 explains its structural role.

### 2. Decode a fixed little-endian integer in JShell

```shell
jshell
```

Then enter:

```java
import java.nio.ByteBuffer;
import java.nio.ByteOrder;
byte[] bytes = {(byte) 0xE8, 0x03, 0x00, 0x00};
ByteBuffer.wrap(bytes).order(ByteOrder.LITTLE_ENDIAN).getInt()
ByteBuffer.wrap(bytes).order(ByteOrder.BIG_ENDIAN).getInt()
```

The first result should be 1,000. The second is different because the same
bytes were interpreted under a different contract.

Exit with:

```text
/exit
```

### 3. Trace the varint cursor

Open
[`ThriftCompactReader.readVarint`](../../core/src/main/java/dev/hardwood/internal/thrift/ThriftCompactReader.java)
and trace input `AC 02` on paper:

| step | byte | payload | shift | accumulated result | stop? |
|---:|---:|---:|---:|---:|---|
| 1 | `AC` | `2C` = 44 | 0 | 44 | no |
| 2 | `02` | 2 | 7 | 300 | yes |

Identify the lines that:

- avoid Java sign extension;
- remove the continuation bit;
- place the payload;
- reject a representation longer than ten bytes.

### 4. Run focused binary-reader tests

```shell
timeout 180s ./mvnw -pl core \
  -Dtest=ThriftCompactReaderTest,ThriftCompactConstantsTest test
```

Choose one rejection test and explain what incorrect outcome its bound prevents.
For example, an eleventh varint byte would make Java reuse low shift positions,
potentially producing a plausible but wrong number.

### 5. Compute widths

Without running code, predict `LevelEncoder.bitWidth` for maxima:

```text
0, 1, 2, 3, 7, 8, 255, 256
```

Then inspect the method and check your results:

```text
0, 1, 2, 2, 3, 4, 8, 9
```

## Common misconceptions

**“Hexadecimal changes the data.”**  
No. Hex is a compact way to print bit patterns.

**“Java `byte` is an unsigned file byte.”**  
No. Mask with `& 0xFF` before unsigned arithmetic.

**“Little-endian means reading bits backward.”**  
No. It orders bytes within a multi-byte value. A bit-packed format separately
defines bit order.

**“All numbers in Parquet are little-endian.”**  
No. Representation depends on context: fixed-width little-endian, varint,
ZigZag-plus-varint, and bit-packed forms all occur.

**“ZigZag is compression.”**  
It is a reversible signed-to-unsigned mapping. A following varint makes values
near zero compact.

**“A decoder can read first and bounds-check afterward.”**  
That risks an uncontrolled exception, allocation, or cursor shift. Validate
before consuming or allocating whenever possible.

## Recap and debug checklist

- What representation does this field use?
- What is the cursor offset before the read?
- Is the value fixed-width, terminated, or length-prefixed?
- If fixed-width, what byte order applies?
- If varint, did a byte with continuation clear appear within the limit?
- If signed, is ZigZag part of this contract?
- Were Java bytes masked before unsigned arithmetic?
- What is the cursor offset after the read?
- Was a file-supplied length checked before allocation or slicing?

## Quiz

1. Convert byte `0xD6` to an eight-bit binary pattern and to an unsigned
   decimal value. What numeric value does the same pattern have as a Java
   `byte`?
2. Decode the four bytes `34 12 00 00` as a little-endian 32-bit integer. Show
   the place-value calculation.
3. Decode unsigned varint `AC 02`, showing each seven-bit contribution and the
   stopping condition.
4. What unsigned numbers do ZigZag encode for `-3` and `3`? What signed number
   does ZigZag value `7` decode to?
5. Compute the minimum bit width for maxima 0, 1, 6, 7, and 8.
6. Trace this Java code. What are `wrong` and `right`, and why?

   ```java
   byte raw = (byte) 0xE8;
   int wrong = raw;
   int right = raw & 0xFF;
   ```

7. A decoder is at offset 40 in a 44-byte buffer. A field declares a
   five-byte body. Should the decoder accept it? Express the check as a
   half-open range.
8. Classify each as a **format rule** or a **Hardwood choice**:
   (a) the tail metadata length is four-byte little-endian; (b)
   `ThriftCompactReader` stores a field header in one `int`; (c) Thrift Compact
   signed `i32` uses ZigZag and varint; (d) malformed binary lengths become
   Java `IOException`s with Hardwood's wording.

Compare your answers with
[`../answers/02-binary-toolkit.md`](../answers/02-binary-toolkit.md).
