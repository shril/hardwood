# Answers — Chapter 4: The Footer and Thrift Compact Protocol

Return to
[`../chapters/04-footer-and-thrift.md`](../chapters/04-footer-and-thrift.md).

## 1. Locate the metadata

Decode `C0 03 00 00` as little-endian:

```text
0xC0 × 256^0 = 192
0x03 × 256^1 = 768
metadata length       = 960 bytes
```

The final length and magic occupy eight bytes, so:

```text
metadataStart = 10,000 - 8 - 960
              = 9,032
metadataEnd   = 10,000 - 8
              = 9,992
```

The half-open range is:

```text
[9,032, 9,992)
```

Its length check is `9,992 - 9,032 = 960`.

## 2. Decode `15 04`

Header `0x15` splits into:

```text
high nibble 1 -> field-id delta 1
low nibble  5 -> i32
```

At the beginning of the struct the previous id is zero, so the field id is:

```text
0 + 1 = 1
```

Payload `04` is unsigned varint 4. ZigZag decoding gives:

```text
(4 >>> 1) ^ -(4 & 1)
= 2 ^ 0
= 2
```

The result is **field 1, wire type i32, signed value 2**. In
`FileMetaData`, that means `version = 2`.

## 3. Decode `19 3C`

The previous `FileMetaData` field id was 1.

```text
0x19: delta 1, wire type LIST -> field id 2
0x3C: short list size 3, element type STRUCT
```

It announces `FileMetaData.schema`, a list of **three `SchemaElement`
structs**.

## 4. Decode `18 02 69 64`

The previous field id is 3:

```text
0x18: delta 1, wire type BINARY
field id = 3 + 1 = 4
```

Binary length is unsigned varint `02`, or two bytes. The two bytes are:

```text
69 64 -> UTF-8 "id"
```

The result is **field 4, binary/string, length 2, value `"id"`**.

## 5. Skipping an unknown field

Every field header carries its wire type. Even without knowing the field's
semantic name, a reader can use that type to determine how to consume its
value: one byte, a varint, a length-prefixed binary value, a collection, or a
nested struct through STOP.

Leaving the cursor unchanged would make the unknown field's payload look like
the next field header. Every following id and value could then be
misinterpreted. Extensibility depends on exact skipping, not on pretending the
bytes are absent.

## 6. Hardwood footer trace

The path is:

```text
InputFile.length() / readRange(...)
  -> ParquetMetadataReader
    -> ThriftCompactReader
      -> FileMetaDataReader
        -> nested readers
          -> FileMetaData
```

[`ParquetMetadataReader`](../../core/src/main/java/dev/hardwood/internal/reader/ParquetMetadataReader.java)
handles file framing and range calculation.

[`ThriftCompactReader`](../../core/src/main/java/dev/hardwood/internal/thrift/ThriftCompactReader.java)
decodes primitive Compact constructs and maintains the cursor.

[`FileMetaDataReader`](../../core/src/main/java/dev/hardwood/internal/thrift/FileMetaDataReader.java)
maps top-level field ids and delegates schema elements, row groups, and other
nested structs.

## 7. Impossible footer length

Apply the bootstrap equation:

```text
footerStart = 150 - 8 - 200
            = -58
```

Offset -58 is outside the file and is earlier than the opening four-byte
magic. The file-framing layer—Hardwood's `ParquetMetadataReader`—must reject
the footer length before constructing a Thrift reader. There is no legitimate
metadata range to decode.

## 8. Format rule or Hardwood choice

1. **Format rule:** the four-byte little-endian metadata length is immediately
   before closing magic.
2. **Hardwood choice:** `"footer-body"` is a Hardwood fetch-reason label.
3. **Format rule:** Parquet's metadata structures are serialized with Thrift
   Compact Protocol.
4. **Hardwood choice:** using the particular
   `ThriftCompactReader.skipField` method is Hardwood's implementation of the
   wire-compatible unknown-field behavior. The ability to skip comes from the
   protocol's typed fields; the Java method does not.
