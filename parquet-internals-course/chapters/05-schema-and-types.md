# Chapter 5 — Schema, Physical Types, and Logical Types

The footer bytes in Chapter 4 contained a list of `SchemaElement` structs.
This chapter turns that flattened list into a tree and separates three
independent questions:

1. Is a field required, optional, or repeated?
2. What primitive representation is stored?
3. What meaning should a reader assign to that representation?

Confusing those axes causes errors that look like decoding failures even when
every byte was read correctly.

## Learning objectives

After this chapter, you should be able to:

1. reconstruct a schema tree from the flattened, depth-first element list;
2. enumerate the eight Parquet physical types;
3. distinguish physical type, logical type, and repetition;
4. derive leaf paths and maximum definition/repetition levels from a schema;
5. convert small DATE, TIMESTAMP, STRING, and DECIMAL examples from physical
   values;
6. explain the relationship between modern logical types and legacy converted
   types;
7. trace Hardwood's footer schema into `FileSchema` and `ColumnSchema`; and
8. identify whether a wrong result belongs to physical decode or logical
   conversion.

## Prerequisite recap

From earlier chapters:

- group nodes provide structure; primitive leaves own column chunks;
- a row group contains one chunk per primitive leaf in schema order;
- footer metadata is a Thrift Compact `FileMetaData`;
- `FileMetaData.schema` is a list of `SchemaElement` structs;
- PLAIN integers use little-endian fixed-width bytes, while metadata integers
  use Thrift's ZigZag-varint rules.

Chapter 4 hand-decoded three schema elements:

```text
root "schema", two children
required INT64 "id"
required INT64 "value"
```

The first element's child count is what turns a flat list into a tree.

## Central mental model: shape, storage, meaning

For every field, ask three separate questions:

```text
shape       storage          meaning
----------- ---------------- ---------------------
OPTIONAL    INT32            DATE
REQUIRED    BYTE_ARRAY       STRING
OPTIONAL    INT32            DECIMAL(7,2)
REQUIRED    INT64            no logical annotation
```

- **Repetition** controls shape: required, optional, or repeated.
- **Physical type** controls the primitive on-disk value family.
- **Logical type** interprets that primitive as a domain value.

Then add a fourth concern below those:

- **Encoding** controls how a sequence of physical values is represented in a
  page.

For example:

```text
OPTIONAL + INT32 + DATE + RLE_DICTIONARY
```

is coherent. Optionality carries null structure, `INT32` is the physical
family, DATE means days from the Unix epoch, and dictionary encoding is one
way a page may store the physical integers.

The read direction is:

```text
page bytes
  -> decode encoding
    -> physical Java value
      -> apply logical annotation
        -> domain Java value
```

A timestamp bug can occur in the last step even when the decoded `long` is
correct.

## The physical type set

Parquet deliberately keeps its physical types small:

| physical type | stored primitive |
|---|---|
| `BOOLEAN` | boolean, one bit per value under PLAIN |
| `INT32` | signed 32-bit integer |
| `INT64` | signed 64-bit integer |
| `INT96` | 96-bit value; deprecated legacy use, especially timestamps |
| `FLOAT` | IEEE 754 binary32 |
| `DOUBLE` | IEEE 754 binary64 |
| `BYTE_ARRAY` | variable-length bytes |
| `FIXED_LEN_BYTE_ARRAY` | fixed-width bytes; width comes from schema |

> **Parquet format rule:** These are the primitive storage families. There is
> no physical `STRING`, `DATE`, `UUID`, or 16-bit integer type.

A smaller physical vocabulary lets encoders specialize around a few
representations. Logical annotations recover richer semantics.

Do not infer a logical type from a physical type alone:

```text
INT32 might mean:
- an ordinary signed integer;
- an 8-, 16-, or 32-bit annotated integer;
- days since the epoch (DATE);
- milliseconds since midnight (TIME(MILLIS));
- an unscaled DECIMAL value.
```

The footer annotation distinguishes them.

## SchemaElement: a flattened tree

In `parquet.thrift`, one `SchemaElement` describes either:

- a **group**: no physical `type`, with `num_children`; or
- a **primitive**: a physical `type`, with no children.

The list is depth-first pre-order:

```text
visit node
then visit each child from left to right
```

Example tree:

```text
schema
├── id
├── customer
│   ├── name
│   └── birth_date
├── amount
└── placed_at
```

Flattened:

```text
0 root schema      group, 4 children
1 id               primitive
2 customer         group, 2 children
3 customer.name    primitive
4 customer.birth_date primitive
5 amount           primitive
6 placed_at        primitive
```

Paths are not stored as one full string in each `SchemaElement`. The tree
position produces `customer.name`. Column-chunk metadata later repeats
`path_in_schema` so a chunk identifies the leaf it contains.

### Reconstructing with a cursor

Start a cursor after the root. To build `n` children:

1. consume the next element;
2. if primitive, emit one leaf;
3. if group, recursively consume exactly its `num_children`;
4. repeat until `n` children are complete.

For the list above:

```text
root needs 4:
  consume id                          root child 1
  consume customer; it needs 2:
    consume name                      customer child 1
    consume birth_date                customer child 2
  customer complete                   root child 2
  consume amount                      root child 3
  consume placed_at                   root child 4
root complete; cursor must be at end
```

A child count that drives the cursor past the list is malformed. Extra
unconsumed elements also indicate an inconsistent tree, even if every
individual Thrift struct parsed.

## Repetition and levels

Each non-root node has a repetition:

- `REQUIRED`: exactly one occurrence when its parent is defined;
- `OPTIONAL`: zero or one occurrence;
- `REPEATED`: zero or more occurrences.

Parquet derives maximum levels along each root-to-leaf path:

```text
max definition level
  += 1 for every OPTIONAL or REPEATED node

max repetition level
  += 1 for every REPEATED node
```

A required node adds neither. Definition levels report how far optional or
repeated structure is defined at one encoded position. Repetition levels report
where repeated structure continues. Chapters 7 and 10 develop their value
streams; here we calculate maxima from schema.

For:

```text
OPTIONAL customer
  REQUIRED name
  OPTIONAL birth_date
```

the leaves have:

```text
customer.name:
  optional customer +1
  required name     +0
  max definition = 1
  max repetition = 0

customer.birth_date:
  optional customer   +1
  optional birth_date +1
  max definition = 2
  max repetition = 0
```

That distinction lets a reader tell “customer is null” from “customer exists
but birth_date is null.”

## Logical types

A logical type annotates a primitive or, for structural annotations such as
LIST and MAP, a group. The annotation constrains the legal physical
representation and supplies interpretation parameters.

Common primitive mappings include:

| logical type | required physical representation | interpretation |
|---|---|---|
| `STRING` | `BYTE_ARRAY` | UTF-8 bytes |
| `ENUM` | `BYTE_ARRAY` | UTF-8 enum symbol |
| `DATE` | `INT32` | signed days since 1970-01-01 |
| `TIME(MILLIS, …)` | `INT32` | milliseconds since midnight |
| `TIME(MICROS/NANOS, …)` | `INT64` | units since midnight |
| `TIMESTAMP(unit, …)` | `INT64` | units from the Unix epoch |
| `INTEGER(8/16/32, signedness)` | `INT32` | narrower/signedness semantics |
| `INTEGER(64, signedness)` | `INT64` | 64-bit signedness semantics |
| `DECIMAL(p,s)` | `INT32`, `INT64`, `BYTE_ARRAY`, or `FIXED_LEN_BYTE_ARRAY` | unscaled integer with scale `s` |
| `UUID` | 16-byte `FIXED_LEN_BYTE_ARRAY` | RFC 4122 byte order |
| `FLOAT16` | 2-byte `FIXED_LEN_BYTE_ARRAY` | IEEE binary16, little-endian |
| `JSON` | `BYTE_ARRAY` | UTF-8 JSON |
| `BSON` | `BYTE_ARRAY` | BSON bytes |
| `UNKNOWN`/null type | any physical type | every value is null |

This is not the complete modern logical-type catalog; Parquet also defines
structural and evolving types. The format specification and `parquet.thrift`
remain authoritative.

### DATE

Physical `INT32` value:

```text
1
```

means one signed day after 1970-01-01:

```text
1970-01-02
```

Value `-1` means 1969-12-31. Under PLAIN, physical value 1 is bytes:

```text
01 00 00 00
```

The bytes yield integer 1 before the DATE annotation produces a calendar date.

### STRING

The text `"Ana"` becomes UTF-8 bytes:

```text
41 6e 61
```

Under PLAIN `BYTE_ARRAY`, the value has a four-byte little-endian length:

```text
03 00 00 00 41 6e 61
```

PLAIN decoding yields `byte[]`; STRING interpretation yields Java `String`.

### DECIMAL

`DECIMAL(7,2)` has precision 7 and scale 2. The value `123.45` has unscaled
integer:

```text
123.45 × 10^2 = 12,345
```

Stored as physical `INT32`, PLAIN bytes are:

```text
12,345 = 0x00003039
bytes  = 39 30 00 00
```

The layers are:

```text
39 30 00 00 -> physical int 12,345 -> decimal 123.45
```

Without the annotation, the same physical value means ordinary integer 12,345.

For `BYTE_ARRAY` and `FIXED_LEN_BYTE_ARRAY` decimals, the unscaled integer uses
big-endian two's-complement bytes. That is a logical-type representation rule
and an important exception to casually treating every number-like payload as
little-endian.

Precision bounds the total decimal digits; scale bounds the digits after the
decimal point and must not exceed precision. `INT32` supports decimal precision
up to 9 and `INT64` up to 18.

### TIMESTAMP

`TIMESTAMP(MICROS, isAdjustedToUTC=true)` with physical value 1,500,000 means:

```text
1,500,000 microseconds
= 1 second + 500,000 microseconds
= 1970-01-01T00:00:01.500Z
```

The PLAIN `INT64` bytes are:

```text
60 e3 16 00 00 00 00 00
```

`isAdjustedToUTC=true` describes an instant on the UTC timeline.
`isAdjustedToUTC=false` describes a local wall-clock date and time with no
specific instant. The physical epoch arithmetic is similar, but assigning a
time zone to a local value changes its meaning and is incorrect.

## A complete worked schema

Consider:

```text
message orders {
  required int64 id;
  optional group customer {
    required binary name (STRING);
    optional int32 birth_date (DATE);
  }
  required int32 amount (DECIMAL(7,2));
  optional int64 placed_at (TIMESTAMP(MICROS, UTC));
}
```

### Tree and flattened elements

```text
0 orders          group, 4 children
1 id              REQUIRED INT64
2 customer        OPTIONAL group, 2 children
3 name            REQUIRED BYTE_ARRAY, STRING
4 birth_date      OPTIONAL INT32, DATE
5 amount          REQUIRED INT32, DECIMAL(7,2)
6 placed_at       OPTIONAL INT64, TIMESTAMP(MICROS, UTC)
```

### Leaf columns

| ordinal | path | physical | logical | max def | max rep |
|---:|---|---|---|---:|---:|
| 0 | `id` | `INT64` | none | 0 | 0 |
| 1 | `customer.name` | `BYTE_ARRAY` | `STRING` | 1 | 0 |
| 2 | `customer.birth_date` | `INT32` | `DATE` | 2 | 0 |
| 3 | `amount` | `INT32` | `DECIMAL(7,2)` | 0 | 0 |
| 4 | `placed_at` | `INT64` | `TIMESTAMP(MICROS, UTC)` | 1 | 0 |

There are five chunks per row group, not six: `customer` is a group, not a
leaf.

### One logical row

```text
id = 10
customer = {name = "Ana", birth_date = null}
amount = 123.45
placed_at = 1970-01-01T00:00:01.500Z
```

Physical leaf values include:

```text
id                    -> long 10
customer.name         -> bytes for "Ana"
customer.birth_date   -> no present physical value at this position
amount                -> int 12,345
placed_at             -> long 1,500,000
```

Structural levels distinguish the null date from a null customer. Values are
decoded physically before `String`, `BigDecimal`, and `Instant` conversion.

## Modern logical types and converted types

Older Parquet files use `SchemaElement.converted_type` plus fields such as
`scale` and `precision`. Modern metadata has a `logicalType` union with
parameter structs. Many annotations have corresponding old and new forms; some
newer annotations do not.

> **Parquet format rule:** Readers must understand the backward-compatibility
> mapping defined by the logical-type specification. When both corresponding
> annotations are written, they must agree.

> **Hardwood implementation choice:** While reconstructing a primitive,
> Hardwood uses modern `logicalType` when present and otherwise maps a legacy
> `convertedType`. This produces one effective `LogicalType` for downstream
> code.

An unknown modern logical union member must not erase the physical type. An
older reader can still expose the physical representation if it safely skips
the unknown annotation.

## Validation belongs on both sides of the boundary

A writer controls what it emits and should reject an impossible declaration:

```text
DATE over BYTE_ARRAY
UUID over FIXED_LEN_BYTE_ARRAY(8)
DECIMAL(12,2) over INT32
```

A reader cannot rewrite an existing foreign file. It should retain enough
physical metadata to inspect or diagnose it and fail at a controlled boundary
if the requested interpretation cannot be honored.

> **Hardwood implementation choice:** `LogicalTypeValidator` is used by schema
> construction on the write side. Footer reading remains tolerant enough to
> preserve foreign physical metadata; conversion methods still check the
> physical/logical pairing before producing a domain value.

This is not permission to silently reinterpret an invalid pairing. Lenient
parsing and correct value conversion are different responsibilities.

## Format rules and implementation choices

> **Parquet format rule:** The schema is a depth-first list whose group elements
> carry child counts and whose primitive elements carry physical types.

> **Parquet format rule:** Logical annotations constrain and interpret physical
> representations; they do not replace the physical type or value encoding.

> **Parquet format rule:** Optional and repeated nodes increase maximum
> definition level; repeated nodes increase maximum repetition level.

> **Hardwood implementation choice:** `FileSchema` stores both a tree of
> `SchemaNode` objects and a schema-ordered list of `ColumnSchema` leaves, plus
> a dot-path lookup map.

> **Hardwood implementation choice:** Hardwood models logical annotations as a
> sealed Java interface and returns Java domain objects such as `LocalDate`,
> `Instant`, `LocalDateTime`, `BigDecimal`, and `UUID`.

> **Hardwood implementation choice:** Modern annotations take precedence over
> legacy converted types when Hardwood derives its effective logical type.

## Current Hardwood code tour

1. [`SchemaElement`](../../core/src/main/java/dev/hardwood/metadata/SchemaElement.java)
   is the direct flattened metadata representation. Compare `isGroup()` and
   `isPrimitive()`.
2. [`PhysicalType`](../../core/src/main/java/dev/hardwood/metadata/PhysicalType.java)
   contains the eight primitive families.
3. [`LogicalType`](../../core/src/main/java/dev/hardwood/metadata/LogicalType.java)
   is Hardwood's sealed hierarchy for parameterized and marker annotations.
4. [`FileSchema.fromSchemaElements`](../../core/src/main/java/dev/hardwood/schema/FileSchema.java)
   rebuilds the tree and leaf list together. In `buildChildren`, locate the two
   level formulas and the shared cursor.
5. [`SchemaNode`](../../core/src/main/java/dev/hardwood/schema/SchemaNode.java)
   represents groups and primitives; its group helpers recognize structs,
   lists, maps, and variants.
6. [`ColumnSchema`](../../core/src/main/java/dev/hardwood/schema/ColumnSchema.java)
   stores path, physical type, repetition, fixed length, leaf ordinal, maximum
   levels, and effective logical type.
7. [`LogicalTypeReader`](../../core/src/main/java/dev/hardwood/internal/thrift/LogicalTypeReader.java)
   decodes the Thrift union.
8. [`LogicalTypeValidator`](../../core/src/main/java/dev/hardwood/internal/schema/LogicalTypeValidator.java)
   pins legal write-side physical/logical pairings.
9. [`LogicalTypeConverter`](../../core/src/main/java/dev/hardwood/internal/conversion/LogicalTypeConverter.java)
   converts physical values on read and keeps local versus UTC-adjusted
   timestamps distinct.
10. [`PhysicalValueConverter`](../../core/src/main/java/dev/hardwood/internal/conversion/PhysicalValueConverter.java)
    performs the write-side inverse with range and precision checks.

The path to remember is:

```text
SchemaElement list
  -> FileSchema tree + ordered ColumnSchema leaves
    -> physical decoder chosen by ColumnSchema.type
      -> LogicalTypeConverter chosen by ColumnSchema.logicalType
```

## Guided lab: trace logical values in a checked-in fixture

This lab uses
`core/src/test/resources/logical_types_test.parquet`.

### 1. Establish expected domain values

Open
[`PqRowApiTest`](../../core/src/test/java/dev/hardwood/PqRowApiTest.java) and
locate `testLogicalTypes`.

Record the expected Java types and first-row values for:

```text
id
birth_date
created_at_millis
```

The test expects an integer, `LocalDate.of(1990, 1, 15)`, and
`Instant.parse("2025-01-01T10:30:00Z")`.

### 2. Run the focused end-to-end test

```shell
timeout 180s ./mvnw -pl core \
  -Dtest=PqRowApiTest#testLogicalTypes test
```

The test crosses footer parsing, schema reconstruction, physical decoding,
logical conversion, and row access. A failure here does not by itself identify
which layer is wrong.

### 3. Trace the DATE branch

```shell
rg -n "DateType|convertToDate|ofEpochDay" \
  core/src/main/java/dev/hardwood/internal/conversion/LogicalTypeConverter.java
rg -n "case DATE|effectiveLogicalType" \
  core/src/main/java/dev/hardwood/schema/FileSchema.java
```

Write the two paths by which DATE can become the effective annotation:

```text
modern logicalType union -> DateType
legacy converted_type DATE -> DateType
```

Then identify the physical Java value expected by `convertToDate`.

### 4. Trace schema levels

Inspect the formulas in
[`FileSchema.buildChildren`](../../core/src/main/java/dev/hardwood/schema/FileSchema.java).
Apply them to:

```text
OPTIONAL group profile
  REPEATED group aliases
    OPTIONAL BYTE_ARRAY value (STRING)
```

For leaf `profile.aliases.value`:

```text
max definition = 3
max repetition = 1
```

Reason: all three nodes are non-required, while only `aliases` is repeated.

### 5. Inspect unknown logical-type behavior

Open
[`SchemaElementReaderTest`](../../core/src/test/java/dev/hardwood/internal/thrift/SchemaElementReaderTest.java).
Explain why its expected result preserves `INT32` and name `col` while setting
the unknown logical type to `null`. Relate this to the separation of storage
from meaning.

Optionally run:

```shell
timeout 180s ./mvnw -pl core \
  -Dtest=SchemaElementReaderTest test
```

## Common misconceptions

**“STRING is a Parquet physical type.”**  
No. STRING annotates `BYTE_ARRAY`.

**“Logical types change the page encoding.”**  
No. Encoding first yields physical values; logical conversion gives them
domain meaning.

**“An optional INT32 is a different physical type from a required INT32.”**  
No. Repetition and physical type are separate schema axes.

**“Every group has its own column chunk.”**  
No. Only primitive leaves have chunks.

**“The schema list is flat because Parquet cannot represent nesting.”**  
The list is a depth-first serialization of a tree. Group child counts restore
nesting.

**“INT32 bytes determine whether a value is a DATE.”**  
The logical annotation does. The same physical integer can have several
meanings.

**“A UTC-adjusted and local timestamp are interchangeable.”**  
No. One denotes an instant; the other denotes a wall-clock value without a
specific instant.

**“All decimal payloads are little-endian.”**  
INT32/INT64 physical values follow their PLAIN little-endian rule, while
byte-array decimal unscaled integers are big-endian two's complement.

## Recap and debug checklist

- Did the flattened tree consume exactly the declared children?
- Is this node a group or primitive?
- What is the full leaf path and schema ordinal?
- What do repetition and ancestor repetition imply for maximum levels?
- What physical type did the page decoder produce?
- Is `type_length` present and valid for `FIXED_LEN_BYTE_ARRAY`?
- Is a modern logical type present? Is there only a legacy converted type?
- Is the logical annotation legal for the physical representation?
- For DECIMAL, are precision, scale, signedness, and byte order correct?
- For TIMESTAMP, are unit and `isAdjustedToUTC` correct?
- Is the wrong value already wrong physically, or only after conversion?

## Quiz

1. List all eight Parquet physical types. Which two are byte-array families?
2. Flatten this schema in depth-first order and list its primitive leaf paths:

   ```text
   message m {
     optional group user {
       required int64 id;
       optional binary email (STRING);
     }
     required double score;
   }
   ```

3. For the schema in question 2, compute maximum definition and repetition
   levels for every leaf.
4. PLAIN bytes `39 30 00 00` decode as physical `INT32`. What value do they
   represent with no logical annotation? What value do they represent as
   `DECIMAL(7,2)`? Show the calculation.
5. A DATE column has physical value `-1`. What calendar value does it
   represent, and which layer performs that change?
6. A `TIMESTAMP(MICROS, isAdjustedToUTC=true)` column yields physical long
   1,500,000. Compute the represented instant.
7. Trace Hardwood's annotation choice when a `SchemaElement` has (a) modern
   `DateType` and legacy `UTF8`, and (b) no modern annotation and legacy
   `DATE`. State the effective logical type in each case.
8. Classify each as a **format rule** or a **Hardwood choice**:
   (a) primitive leaves carry physical types; (b) schema lookups use
   dot-separated strings; (c) DATE is signed days since the Unix epoch in
   `INT32`; (d) decoded DATE values are returned as Java `LocalDate`.

Compare your answers with
[`../answers/05-schema-and-types.md`](../answers/05-schema-and-types.md).
