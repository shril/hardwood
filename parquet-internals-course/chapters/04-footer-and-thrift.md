# Chapter 4 — The Footer and Thrift Compact Protocol

Chapter 3 showed that a reader needs metadata to locate chunks. This chapter
explains how it finds and decodes that metadata.

“Read the footer” is really two operations:

1. use a fixed trailer to locate the serialized metadata bytes;
2. interpret those bytes as a Thrift Compact `FileMetaData` struct.

Keeping the framing layer separate from the serialization layer makes failures
much easier to diagnose.

## Learning objectives

After this chapter, you should be able to:

1. calculate a footer's start from file length and trailer bytes;
2. explain why a Parquet reader starts at the end;
3. decode short-form Thrift field and list headers;
4. combine field-id deltas, wire types, ZigZag integers, varints, and STOP
   markers;
5. hand-decode the opening fields of a real checked-in footer;
6. explain how unknown fields can be skipped without losing the cursor;
7. trace Hardwood's footer read into metadata records; and
8. distinguish invalid file framing from invalid Thrift content.

## Prerequisite recap

From Chapter 2:

- four-byte little-endian integers use the first byte for the least
  significant place;
- unsigned varints contribute seven bits per byte;
- ZigZag maps signed values around zero;
- one byte can pack high and low nibbles;
- a reader must know the contract and cursor position.

From Chapter 3:

- a footer describes the schema, row groups, chunks, offsets, sizes, codecs,
  and optional locators;
- optional indexes are separate regions whose offsets may be in metadata;
- the conventional file ends with serialized metadata, a four-byte metadata
  length, and `PAR1`.

## Central mental model: bootstrap, then interpret

The tail is deliberately self-locating:

```text
              footerStart                       fileSize
                   |                                |
                   v                                v
... data ... [serialized FileMetaData] [length LE] [PAR1]
             <------ footerLength ------> <--- 8 --->
```

The bootstrap equation is:

```text
footerStart = fileSize - 8 - footerLength
```

The final eight bytes can be located using only `fileSize`. Once they reveal
`footerLength`, the reader can request exactly the metadata range. Once that
range is decoded, it can request selected column chunks.

Use this two-layer mental model:

```text
file framing
  validates magic and locates metadata bytes
          |
          v
Thrift Compact decoding
  maps numbered fields to FileMetaData
          |
          v
reader planning
  maps selected leaves to chunk and index ranges
```

If the final magic is wrong, Thrift is not the problem. If the range is valid
but a struct never reaches STOP, file framing succeeded and metadata decoding
failed.

## Why metadata is at the end

A writer can stream column chunks forward without knowing final offsets, sizes,
statistics, or row-group counts in advance. At close, it writes metadata that
summarizes what it emitted, then its length and closing magic.

The reader performs the reverse:

1. obtain the file length;
2. validate the opening magic;
3. read the final eight bytes;
4. decode metadata length and validate closing magic;
5. validate the calculated metadata range;
6. read that range;
7. decode `FileMetaData`; and
8. plan data reads.

> **Parquet format rule:** Unencrypted Parquet files use `PAR1` at both ends,
> and store a four-byte little-endian metadata length immediately before the
> closing magic.

Parquet Modular Encryption has encrypted-footer and plaintext-footer modes.
Encrypted-footer files use `PARE` framing, while plaintext-footer files expose
an encryption algorithm field in metadata.

> **Hardwood implementation choice:** Hardwood detects both encrypted forms
> and fails with `EncryptedParquetException`; it does not decrypt them.

## Worked trailer: a checked-in file

The checked-in file
`core/src/test/resources/plain_uncompressed.parquet` is 722 bytes. Its final
eight bytes are:

```text
18 02 00 00  50 41 52 31
```

Decode the first four as little-endian:

```text
0x18 × 256^0 =  24
0x02 × 256^1 = 512
0x00 × 256^2 =   0
0x00 × 256^3 =   0
footerLength     = 536 bytes
```

The next four bytes are ASCII `PAR1`. Therefore:

```text
footerStart = 722 - 8 - 536
            = 178
```

The regions are:

```text
[0, 4)     opening PAR1
[4, 178)   column data and any pre-footer structures
[178, 714) serialized FileMetaData (536 bytes)
[714, 718) little-endian length
[718, 722) closing PAR1
```

Check:

```text
178 + 536 = 714
714 + 4 + 4 = 722
```

The framing calculation says where the metadata is. It says nothing yet about
what fields are inside.

## Thrift IDL versus Thrift Compact bytes

The authoritative `parquet.thrift` file defines structs and numbered fields.
Conceptually, its `FileMetaData` begins with fields like:

```thrift
struct FileMetaData {
  1: required i32 version
  2: required list<SchemaElement> schema
  3: required i64 num_rows
  4: required list<RowGroup> row_groups
  ...
}
```

Names help humans. Field **ids** and wire types identify data on disk.

Thrift Compact Protocol is the serialization contract. The footer contains a
bare serialized `FileMetaData` struct, not a Thrift RPC message with a method
name or protocol envelope.

> **Parquet format rule:** Parquet metadata structures and page headers are
> serialized using Thrift Compact Protocol according to `parquet.thrift`.

## Struct field headers

A short-form Compact field header is one byte:

```text
dddd tttt
```

- high nibble `dddd`: current field id minus previous field id, from 1 to 15;
- low nibble `tttt`: wire type.

Useful wire codes are:

| low nibble | wire type |
|---:|---|
| `0` | STOP, when the whole byte is zero |
| `1` | boolean true |
| `2` | boolean false |
| `3` | byte |
| `4` | i16 |
| `5` | i32 |
| `6` | i64 |
| `7` | double |
| `8` | binary/string |
| `9` | list |
| `A` | set |
| `B` | map |
| `C` | struct |

If the delta is zero, the actual field id follows separately as a
ZigZag-varint `i16`. Structs end with byte `00`, the STOP marker.

Field-id delta state belongs to one struct. Entering a nested struct resets its
previous id to zero; leaving restores the outer context.

### Worked header

Given header `0x15` at the beginning of a struct:

```text
high nibble 1 -> field id = previous 0 + 1 = 1
low nibble  5 -> wire type i32
```

If its payload byte is `04`:

```text
unsigned varint = 4
ZigZag decode   = 2
```

For `FileMetaData`, this is `version = 2`.

## Collection and binary headers

A short-form list header is also one byte:

```text
ssss tttt
```

- high nibble `ssss`: element count from 0 to 14;
- low nibble `tttt`: element wire type.

For 15 or more elements, high nibble `F` signals that an unsigned varint count
follows.

Thus `0x3C` means:

```text
3 elements, each a struct
```

A binary or string value is:

```text
[unsigned varint byte length][that many bytes]
```

Strings use UTF-8 and have no terminating zero byte.

Counts and binary lengths use unsigned varints, not ZigZag. Signed `i16`,
`i32`, and `i64` values use ZigZag followed by an unsigned varint. Doubles are
fixed eight-byte little-endian values. Boolean struct fields carry true or
false in the header's type nibble and have no separate payload.

## Hand-decoding the real footer

At calculated offset 178, the first bytes of the fixture's footer are:

```text
15 04 19 3c 35 00 18 06 73 63 68 65 6d 61 15 04 00
15 04 25 00 18 02 69 64 00 ...
```

### `FileMetaData.version`

```text
15 -> field delta 1, i32 -> field 1
04 -> ZigZag-varint 4 -> signed value 2
```

So:

```text
version = 2
```

### `FileMetaData.schema`

The next header is:

```text
19 -> field delta 1 from id 1, list -> field 2
3c -> list size 3, element type struct
```

The footer has three flattened `SchemaElement` structs: a root and two leaves.
Chapter 5 reconstructs the tree.

### First `SchemaElement`: the root

A list element is a struct body, so its field-id context begins at zero:

```text
35 00
18 06 73 63 68 65 6d 61
15 04
00
```

Decode:

1. `35`: delta 3, i32 → field 3, `repetition_type`.
2. payload `00`: ZigZag 0 → enum value 0, `REQUIRED`.
3. `18`: delta 1, binary → field 4, `name`.
4. length `06`, then UTF-8 bytes `73 63 68 65 6d 61` → `"schema"`.
5. `15`: delta 1, i32 → field 5, `num_children`.
6. payload `04`: ZigZag 4 → integer 2.
7. `00`: STOP for this `SchemaElement`.

The root is therefore named `schema` and has two children.

### Second `SchemaElement`: leaf `id`

After the root's STOP, the next struct starts with fresh field-id state:

```text
15 04 25 00 18 02 69 64 00
```

- `15 04`: field 1 `type`; ZigZag 4 → enum value 2, Parquet `INT64`.
- `25 00`: delta 2 → field 3 `repetition_type`; enum 0, `REQUIRED`.
- `18 02 69 64`: field 4 binary, length 2, UTF-8 `"id"`.
- `00`: STOP.

This short example exercises field deltas, type codes, ZigZag, varints, binary
lengths, nested context, enum lookup, and STOP.

## Skipping unknown fields

Thrift fields carry a wire type, so a reader that does not recognize an id can
consume its value according to that type:

```text
unknown binary -> read length, skip bytes
unknown list   -> read element count/type, skip each element
unknown struct -> process fields recursively until STOP
```

That supports metadata evolution: a newer writer can add an optional field and
an older reader can continue with later known fields.

Skipping must be exact. Treating a list of booleans as struct boolean fields,
for example, would consume the wrong number of bytes because collection
booleans have payload bytes while boolean fields put their value in the field
header.

A known field with the wrong wire type is also dangerous. Hardwood's readers
use the declared type to skip the value rather than decode it under the
expected type. Required collections whose element type is wrong are rejected,
because pretending that their content is the expected struct would
desynchronize the cursor.

## Defensive metadata decoding

The footer is untrusted input. A few bytes can claim:

- a multi-gigabyte binary value;
- billions of list elements;
- a negative page size after ZigZag decode;
- an offset outside the file;
- an unterminated struct or varint.

The safe order is:

```text
read declaration -> validate against bounds/invariants -> allocate or slice
```

not:

```text
read declaration -> allocate -> discover truncation later
```

This is both correctness and security. Truncating an oversized length to Java
`int` might return plausible, wrong metadata and leave the cursor in the middle
of the field.

## Format rules and implementation choices

> **Parquet format rule:** The trailer layout and Thrift Compact serialization
> determine interoperable bytes.

> **Parquet format rule:** Field ids, not source-language field names, identify
> Thrift struct fields. Unknown fields can be skipped by their declared wire
> type.

> **Hardwood implementation choice:** `ParquetMetadataReader` performs separate
> range reads for opening magic, final trailer, and footer body, each tagged
> with a fetch reason.

> **Hardwood implementation choice:** `ThriftCompactReader` uses a sliced
> `ByteBuffer`, packs a decoded field id and type into one Java `int`, and keeps
> field-id context as mutable reader state.

> **Hardwood implementation choice:** Hardwood validates binary and collection
> declarations against remaining bytes before allocation and reports malformed
> metadata through controlled I/O failures with file context.

> **Hardwood implementation choice:** Dedicated readers such as
> `FileMetaDataReader` and `SchemaElementReader` hand-map Thrift field ids to
> immutable metadata records.

## Current Hardwood code tour

1. [`ParquetMetadataReader`](../../core/src/main/java/dev/hardwood/internal/reader/ParquetMetadataReader.java)
   implements the file-framing bootstrap. Trace `fileSize`, `footerInfoPos`,
   `footerLength`, and `footerStart`.
2. [`ThriftCompactConstants`](../../core/src/main/java/dev/hardwood/internal/thrift/ThriftCompactConstants.java)
   anchors the STOP marker and wire type codes.
3. [`ThriftCompactReader`](../../core/src/main/java/dev/hardwood/internal/thrift/ThriftCompactReader.java)
   implements varints, ZigZag, field headers, list headers, strings, bounds, and
   recursive skipping.
4. [`FileMetaDataReader`](../../core/src/main/java/dev/hardwood/internal/thrift/FileMetaDataReader.java)
   dispatches field ids 1–8 and delegates nested structures.
5. [`SchemaElementReader`](../../core/src/main/java/dev/hardwood/internal/thrift/SchemaElementReader.java)
   maps schema field ids and calls `LogicalTypeReader` for field 10.
6. [`MalformedMetadataValidationTest`](../../core/src/test/java/dev/hardwood/internal/thrift/MalformedMetadataValidationTest.java)
   uses hand-built bytes to pin fail-early behavior.
7. [`ThriftCompactConstantsTest`](../../core/src/test/java/dev/hardwood/internal/thrift/ThriftCompactConstantsTest.java)
   writes the normative wire codes literally so production and test helpers
   cannot drift together unnoticed.

The complete opening path is:

```text
InputFile.length/readRange
  -> ParquetMetadataReader
    -> ThriftCompactReader
      -> FileMetaDataReader
        -> SchemaElementReader / RowGroupReader / ...
          -> FileMetaData record
```

## Guided lab: locate and begin decoding a real footer

Run from the repository root.

### 1. Reproduce the trailer calculation

```shell
FILE=core/src/test/resources/plain_uncompressed.parquet
SIZE=$(stat -c %s "$FILE")
od -An -tx1 -j $((SIZE-8)) -N8 "$FILE"
```

Record:

```text
SIZE = 722
tail = 18 02 00 00 50 41 52 31
```

Calculate 536 and footer offset 178 by hand before continuing.

### 2. Display the footer's opening bytes

```shell
od -An -tx1 -j 178 -N32 "$FILE"
```

Annotate at least these segments:

```text
15 04       FileMetaData field 1, version 2
19 3c       field 2, list of three structs
35 00       first SchemaElement field 3, REQUIRED
18 06 ...   field 4, six-byte name "schema"
15 04       field 5, two children
00          first SchemaElement STOP
```

Do not search for byte `00` globally to find every struct end; zero can occur
inside payloads. STOP has meaning only where the cursor expects a field header.

### 3. Match bytes to source

```shell
rg -n "footerInfoPos|footerLength|footerStart|FileMetaDataReader" \
  core/src/main/java/dev/hardwood/internal/reader/ParquetMetadataReader.java
rg -n "case 1:|case 2:|case 3:|case 4:" \
  core/src/main/java/dev/hardwood/internal/thrift/FileMetaDataReader.java
```

For each arithmetic line in `ParquetMetadataReader`, identify whether it is a
format rule, a bounds check, or a Hardwood diagnostic choice.

### 4. Trace one malformed declaration

Open
[`MalformedMetadataValidationTest`](../../core/src/test/java/dev/hardwood/internal/thrift/MalformedMetadataValidationTest.java)
and choose either `oversizedBinaryLengthRejected` or
`requiredListOfWrongElementTypeRejected`.

Write down:

1. what the bytes claim;
2. what a naive decoder might do;
3. where Hardwood rejects it; and
4. what cursor or allocation failure is prevented.

Run the class:

```shell
timeout 180s ./mvnw -pl core \
  -Dtest=MalformedMetadataValidationTest test
```

### 5. Check nested field-id state

Inspect `pushFieldIdContext()` and `popFieldIdContext()` in
[`ThriftCompactReader`](../../core/src/main/java/dev/hardwood/internal/thrift/ThriftCompactReader.java).
Explain why the first header of leaf `id` is again `15`, even though the prior
root element ended at field id 5. Each list element is a new struct, so its
first field delta starts from zero.

## Common misconceptions

**“The last four bytes are the footer length.”**  
They are closing magic. The preceding four bytes are metadata length.

**“A reader must scan pages to find the footer.”**  
No. File length and the fixed eight-byte trailer locate it directly.

**“The footer starts with a Thrift protocol magic number.”**  
No. It is a bare Compact-encoded `FileMetaData` struct.

**“Thrift field names are stored in the footer.”**  
No. Numeric field ids and wire types are stored. Names come from the IDL known
to the reader.

**“Every Thrift integer is a varint of the signed value.”**  
Signed `i16/i32/i64` values are ZigZag-mapped first. Unsigned lengths and counts
are not.

**“Byte `00` always means the entire footer is over.”**  
STOP ends the struct whose field header is currently expected. Nested structs
have their own STOP markers.

**“Unknown fields can simply be ignored without reading them.”**  
Their bytes must be skipped according to their declared type to leave the
cursor at the next field.

## Recap and debug checklist

- Are opening and closing magic valid?
- Did the four bytes before closing magic decode little-endian?
- Is `footerStart = fileSize - 8 - footerLength` inside `[4, fileSize - 8]`?
- Does the metadata range contain exactly `footerLength` bytes?
- At the failing cursor, is a field header expected or a payload?
- What are the field-id delta and wire type?
- Does the known field expect that wire type?
- Was a signed value ZigZag-decoded?
- Was a length/count checked before allocation?
- Was nested field-id context reset and restored?
- Did every struct reach STOP?

## Quiz

1. A 10,000-byte unencrypted file ends with `C0 03 00 00 50 41 52 31`.
   Compute the metadata length, metadata start, and half-open metadata range.
2. At the start of `FileMetaData`, decode bytes `15 04`: field id, wire type,
   and signed value.
3. Immediately afterward, decode `19 3C`. What field id is this, and what
   collection does it announce?
4. Within a `SchemaElement`, the previous field id is 3. Decode
   `18 02 69 64`: field id, wire type, length, and value.
5. Why can a reader skip a new optional Thrift field it does not understand,
   and why is simply leaving the cursor unchanged incorrect?
6. Trace the Hardwood methods from an `InputFile` to a populated
   `FileMetaData`, naming the framing class, primitive protocol reader, and
   top-level struct reader.
7. A 150-byte file claims a footer length of 200. Compute `footerStart` and
   explain which layer should reject it before Thrift decoding.
8. Classify each as a **format rule** or a **Hardwood choice**:
   (a) metadata length immediately precedes closing magic; (b) footer-body
   fetches carry the reason string `"footer-body"`; (c) metadata is Thrift
   Compact; (d) an unknown field is skipped through
   `ThriftCompactReader.skipField`.

Compare your answers with
[`../answers/04-footer-and-thrift.md`](../answers/04-footer-and-thrift.md).
