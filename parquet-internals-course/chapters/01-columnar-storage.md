# Chapter 1 — From SQL Tables to Columnar Storage

Parquet is easiest to understand by starting with a query, not with bytes. A SQL
engine sees rows, but many analytical queries touch only a few fields from each
row. Parquet arranges data so a reader can avoid fetching and decoding the other
fields.

This chapter develops that idea without assuming any knowledge of hexadecimal
notation, byte order, compression, or binary encodings. Chapter 2 introduces
those tools.

## Learning objectives

After this chapter, you should be able to:

1. contrast row-oriented and column-oriented storage for an analytical query;
2. explain why storing each column separately enables projection;
3. distinguish a logical table, an on-disk leaf column, and a Java output
   array;
4. explain why columnar layout helps compression without claiming that
   columnar layout *is* compression;
5. identify the Hardwood classes that express range reads and column
   projection; and
6. choose the first layer to inspect when an unrequested column is read.

## Prerequisite recap

You need only the course prerequisites:

- a table consists of named columns and rows;
- `SELECT id, city FROM people` requests two columns;
- `NULL` means a value is absent, not zero or an empty string;
- Java arrays such as `long[]` hold values of one type.

One SQL distinction matters throughout the course:

```sql
SELECT city FROM people;
```

is **projection**: it chooses a column. This is different from:

```sql
SELECT * FROM people WHERE city = 'Oslo';
```

which uses a **predicate** to choose rows. A reader can apply both, but the
layout reasons for the two optimizations are different. This chapter focuses on
projection.

## Central mental model: rotate, keep coordinates, rotate back

Imagine rotating a table so values from the same column become adjacent:

```text
logical rows                     column-oriented streams

id   city    score               id:     10, 11, 12, 13
10   Oslo       8                city:   Oslo, Rome, Oslo, NULL
11   Rome       5       --->     score:   8,    5,    9,    6
12   Oslo       9
13   NULL       6
```

The values have not changed. Their **storage neighborhood** has changed.
Coordinates are needed to reconstruct which value belongs to which record and
where nulls occur. For a flat required column, position is enough: the third
`id` and third `score` both belong to row 3. Optional and nested data require
additional structural information; later chapters call it definition and
repetition levels.

Keep this pipeline in mind:

```text
logical records
      |
      | split by schema leaf
      v
independent leaf-column streams
      |
      | store locations and structure in metadata
      v
selected byte ranges
      |
      | decode only requested leaves
      v
typed Java arrays
      |
      | optionally assemble records
      v
logical records
```

The central rule is:

> A Parquet reader plans work by leaf column, even when its caller eventually
> consumes rows.

Groups such as a struct organize the schema. Primitive leaves carry value
streams. If `address` contains `city` and `zip`, the stored leaves are paths
such as `address.city` and `address.zip`; there is no separate value stream for
the `address` group itself.

## A worked data example

Consider this four-row table:

| row | `order_id` | `region` | `amount` |
|---:|---:|---|---:|
| 0 | 700 | EU | 25 |
| 1 | 701 | US | 40 |
| 2 | 702 | EU | 30 |
| 3 | 703 | EU | 50 |

Suppose, only for this sizing exercise, that each stored `order_id` occupies 8
units, each `region` occupies 2, and each `amount` occupies 8. Ignore headers
and compression.

A row-oriented arrangement interleaves fields:

```text
[700, EU, 25] [701, US, 40] [702, EU, 30] [703, EU, 50]
```

A column-oriented arrangement groups fields:

```text
[700, 701, 702, 703] [EU, US, EU, EU] [25, 40, 30, 50]
```

Both arrangements contain:

```text
4 × (8 + 2 + 8) = 72 units
```

Now evaluate:

```sql
SELECT amount FROM orders;
```

If a row-oriented reader cannot jump over individual fields, it reads all 72
units and discards 40 units of `order_id` and `region` data. With locations for
separate column ranges, a column-oriented reader needs the four `amount`
values:

```text
4 × 8 = 32 units
```

The avoided value data is:

```text
72 - 32 = 40 units
```

Real Parquet files also require metadata, page headers, null structure, and
possibly dictionaries. The example demonstrates the direction of the saving,
not an exact file-size formula.

Now change the query:

```sql
SELECT amount FROM orders WHERE region = 'EU';
```

The result projects `amount`, but evaluating the predicate also needs
`region`, unless metadata can prove that a larger region contains no matching
rows. A safe initial read plan therefore includes both leaf columns:

```text
requested output leaf: amount
predicate leaf:        region
planned leaves:        amount, region
```

Reading `order_id` would still be unnecessary. This distinction becomes useful
when debugging: a column absent from the output can be legitimately read
because a filter uses it.

### Why adjacent values often compress well

The `region` stream is:

```text
EU, US, EU, EU
```

It has a small vocabulary and repetition. A representation can store each
distinct region once and encode the sequence with small references. A
row-oriented stream repeatedly interrupts regions with unrelated integers,
making type-specific representations harder to apply over a large run.

Columnar layout does not guarantee a smaller file. Random byte strings may
compress poorly, and metadata adds overhead. The layout merely gives encoders
and compressors homogeneous input and gives readers independent ranges.

## Format rules and implementation choices

The following labels will appear throughout the course.

> **Parquet format rule:** A row group has one column chunk for each leaf
> column. Values for one leaf within that row group are stored together in its
> column chunk. File metadata describes the chunks and their locations.

> **Parquet format rule:** Group schema nodes describe structure; primitive
> schema nodes are the stored columns.

> **Parquet format rule:** Parquet defines an interoperable on-disk
> representation. It does not prescribe a Java reader API, a batch size, or a
> particular I/O backend.

> **Hardwood implementation choice:** Hardwood represents an input as
> `InputFile` and asks it for a byte range with `readRange(offset, length)`.
> Local files, in-memory buffers, and object-store implementations can satisfy
> the same contract.

> **Hardwood implementation choice:** Callers express projection through
> `ColumnProjection`, and Hardwood exposes both row-oriented and
> column-oriented reader APIs. Returning `long[]` from a `ColumnReader` is an
> API choice, not part of the Parquet format.

> **Hardwood implementation choice:** A filter may augment the requested
> projection with predicate columns. Those extra columns participate in
> evaluation but need not appear in the caller's payload.

This separation prevents a common category error: “Parquet returns a `long[]`”
is false. Parquet stores a typed column representation; Hardwood chooses a
primitive Java array at one reader boundary.

## Three kinds of “column”

The word *column* is overloaded. Use a qualifier when reasoning:

1. **SQL column** — a named field in a logical relation, such as `amount`.
2. **Parquet leaf column** — a primitive path in a schema tree, such as
   `customer.address.zip`.
3. **Output column** — a runtime object or primitive array made available by a
   reader.

For a flat table these often line up one-to-one. For nested data they need not:
a top-level SQL `address` struct maps to several Parquet leaf columns and may
be assembled into one row object.

Likewise, a row is not an elementary on-disk Parquet object. A row is a logical
coordinate reconstructed from column positions and nesting information.

## Current Hardwood code tour

Follow these links in order:

1. [`InputFile`](../../core/src/main/java/dev/hardwood/InputFile.java) is the
   storage boundary. Notice `length()` and `readRange(long, int)`. The
   interface requires implementations to be safe for concurrent use after
   opening.
2. [`ColumnProjection`](../../core/src/main/java/dev/hardwood/schema/ColumnProjection.java)
   stores either “all columns” or a set of names. Its comments show flat and
   dot-separated nested paths.
3. [`ProjectedSchema`](../../core/src/main/java/dev/hardwood/internal/schema/ProjectedSchema.java)
   resolves those names against a file schema. It is where a user-facing name
   becomes a concrete set of leaves.
4. [`ParquetFileReader`](../../core/src/main/java/dev/hardwood/reader/ParquetFileReader.java)
   offers `rowReader()`, `columnReader(...)`, and
   `columnReaders(ColumnProjection)`. These are different consumption models
   over the same file layout.
5. [`ColumnReader`](../../core/src/main/java/dev/hardwood/reader/ColumnReader.java)
   exposes typed batch accessors. Search for `getLongs()` and note that values,
   validity, and nested layers are separate concerns.
6. [`ParquetReaderTest`](../../core/src/test/java/dev/hardwood/ParquetReaderTest.java)
   gives a small executable example. The
   `testReadPlainParquet` fixture has required `id` and `value` leaves and
   asserts the decoded primitive arrays.

Do not try to understand the full planning pipeline yet. At this point, the
important path is:

```text
ColumnProjection
    -> ProjectedSchema (selected leaf paths)
    -> planned InputFile ranges
    -> ColumnReader typed batch
```

Later chapters put concrete file regions and decoders into the middle.

## Guided lab: observe projection at the source boundary

Run this lab from the repository root. It uses a checked-in fixture and focused
source inspection. It does not require understanding the fixture's bytes.

### 1. Establish the fixture's logical facts

Open
[`ParquetReaderTest`](../../core/src/test/java/dev/hardwood/ParquetReaderTest.java)
and locate `testReadPlainParquet`.

Record:

- the fixture path;
- its row count;
- its two leaf names and physical types;
- the three expected values in each leaf.

You should find `core/src/test/resources/plain_uncompressed.parquet`, three
rows, and two required 64-bit integer leaves named `id` and `value`.

### 2. Run the focused read test

```shell
timeout 180s ./mvnw -pl core -Dtest=ParquetReaderTest#testReadPlainParquet test
```

This proves that both logical columns can be reconstructed independently. A
passing test does not yet prove that projection avoided all unrelated I/O; that
requires an observable input boundary.

### 3. Find that boundary

```shell
rg -n "readRange|length\\(\\)" core/src/main/java/dev/hardwood/InputFile.java
rg -n "class CountingInputFile|readRange" \
  core/src/test/java/dev/hardwood/internal/reader/CountingInputFile.java
```

Inspect
[`CountingInputFile`](../../core/src/test/java/dev/hardwood/internal/reader/CountingInputFile.java).
Answer:

- What method can a test intercept to count value-data reads?
- What two numbers identify a requested range?
- Why would counting returned rows be insufficient evidence of projection?

The last answer is important: a reader could fetch the entire file, discard one
column, and still return the correct projected rows.

### 4. Trace a projection name to selected leaves

```shell
rg -n "projectsAll|getProjectedColumnNames" \
  core/src/main/java/dev/hardwood/internal/schema/ProjectedSchema.java
rg -n "augmentWithPredicateColumns" \
  core/src/main/java/dev/hardwood/reader/ParquetFileReader.java
```

Read the surrounding methods. Sketch two plans:

```text
projection = columns("value"), no predicate
selected leaves = ?

projection = columns("value"), predicate uses "id"
selected leaves for evaluation = ?
payload leaves returned = ?
```

Expected sketch:

```text
value

value + id
value
```

### 5. State an I/O assertion

Without writing a test, formulate one useful assertion for a projection test.
It should be about ranges or bytes requested through `InputFile`, not only
about returned values. For example:

> After footer reads, reading only `value` must not request a range wholly
> inside the `id` column chunk.

Chapter 3 supplies the vocabulary needed to make that assertion precise.

## Common misconceptions

**“Columnar means the file has one copy of each distinct value.”**  
No. Grouping values by column is layout. A dictionary is one possible encoding
inside that layout.

**“Projection and filtering are the same optimization.”**  
Projection removes unneeded columns. Filtering removes rows. A predicate column
may be read even when it is not returned.

**“A Parquet column is always a top-level SQL column.”**  
No. Parquet stores primitive leaves. One nested top-level field can have many
leaf paths.

**“If the result is correct, projection worked.”**  
Correct output says nothing about avoided I/O. Projection is an efficiency
property and must be observed at the range-read or planning boundary.

**“Columnar files contain no rows.”**  
Rows remain the logical coordinate system. Row groups count rows, and a reader
can assemble rows; values simply are not stored record-by-record.

**“Columnar layout always makes a file smaller.”**  
No. It creates good conditions for type-specific encoding and compression, but
data distribution and metadata overhead determine the actual size.

## Recap and debug checklist

Keep this short checklist:

- What does the caller project?
- Does a predicate add hidden working columns?
- What are the selected primitive leaf paths?
- Which file ranges correspond to those leaves?
- Did `InputFile.readRange` request an unselected chunk?
- Are decoded primitive arrays correct before row assembly?
- Am I confusing layout, encoding, and compression?

The mental model so far is:

```text
rows -> leaf streams -> selected ranges -> typed arrays -> optional rows
```

## Quiz

Write down your reasoning, not just the final answer.

1. In one sentence each, define projection and filtering.
2. A flat table has 1,000 rows and three fixed-width columns occupying 8, 4,
   and 16 bytes per row. Ignoring all overhead and compression, how many value
   bytes are stored? How many are needed for a projection of only the 4-byte
   column? What fraction of the value bytes is avoided?
3. A query returns `name` but filters on `age`. Which leaf columns must a
   straightforward correct reader decode? Which one appears in the output?
4. The schema has a group `address` with primitive children `city` and `zip`.
   How many stored leaf columns does that describe, and what are their paths?
5. A test projects one column and receives exactly the expected values. Explain
   why this does not prove that physical projection occurred, and name the
   Hardwood boundary that can provide stronger evidence.
6. Trace this call conceptually:
   `ColumnProjection.columns("address.city")`. Which class holds the requested
   name, which class resolves it to leaves, and which interface ultimately
   serves planned bytes?
7. Classify each statement as a **format rule** or a **Hardwood choice**:
   (a) one column chunk holds one leaf's data for a row group; (b) decoded
   `INT64` values are exposed as `long[]`; (c) primitive schema leaves carry
   stored values; (d) an input backend implements `readRange(long, int)`.

Compare your answers with
[`../answers/01-columnar-storage.md`](../answers/01-columnar-storage.md).
