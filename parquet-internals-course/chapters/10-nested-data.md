# Chapter 10 — Nested Data and Dremel Levels

Allow 45–75 minutes. This chapter explains how one primitive leaf stream
retains null containers, empty containers, null elements, and row boundaries.

## Objectives

After this chapter you should be able to:

1. derive maximum definition and repetition levels from a list schema;
2. encode null list, empty list, null element, and present elements distinctly;
3. distinguish level-entry count, real leaf-slot count, and dense value count;
4. use repetition levels to recover top-level record boundaries;
5. explain page-boundary continuation of a repeated record; and
6. map Dremel levels to Hardwood's current STRUCT/REPEATED layer model.

## Prerequisite recap

Chapter 5 introduced `REQUIRED`, `OPTIONAL`, and `REPEATED`. Chapter 7 showed
that a value is present only when its definition reaches the leaf maximum.
Chapter 8 provided the hybrid encoding used for levels.

The standard three-level list shape is:

```text
optional group tags (LIST) {
  repeated group list {
    optional binary element (STRING);
  }
}
```

Walk from root to `element`:

- optional `tags` raises maximum definition to 1;
- repeated `list` raises maximum definition to 2 and repetition to 1;
- optional `element` raises maximum definition to 3.

Thus the leaf has:

```text
max definition = 3
max repetition = 1
```

## Central mental model: each level pair is a breadcrumb

Each raw leaf position carries:

```text
(repetition level, definition level, maybe a dense value)
```

Definition answers “how far down the schema path exists?” Repetition answers
“at which repeated depth does this position continue the preceding record?”

For the list schema above:

| Definition | Meaning |
|---:|---|
| 0 | `tags` itself is null |
| 1 | `tags` is present, but has no repeated item: empty list |
| 2 | a list item exists, but optional `element` is null |
| 3 | `element` is present and consumes one dense value |

For maximum repetition 1:

| Repetition | Meaning |
|---:|---|
| 0 | start a new top-level record |
| 1 | continue the same top-level record with another list element |

**Parquet format rule:** repetition and definition levels are interpreted
against the schema path. The integer `1` has no universal “empty list” meaning
outside that path.

## Hand-worked four-state example

Encode four rows:

```text
row 0: null
row 1: []
row 2: [null]
row 3: ["A", "B"]
```

The raw Dremel sequence is:

| Raw position | Row | Repetition | Definition | Dense value |
|---:|---:|---:|---:|---|
| 0 | 0 | 0 | 0 | none |
| 1 | 1 | 0 | 1 | none |
| 2 | 2 | 0 | 2 | none |
| 3 | 3 | 0 | 3 | `"A"` |
| 4 | 3 | 1 | 3 | `"B"` |

There is at least one raw position for each top-level row, even a null or empty
list. That placeholder is how the column stream preserves row alignment.

The three important counts are:

```text
top-level rows       = 4
level/raw positions  = 5
real element slots   = 3   # null element, "A", "B"
dense encoded values = 2   # "A", "B"
```

The null-list and empty-list positions are phantom leaf positions: they
preserve structure but do not represent actual element slots. `[null]` does
contain one real element slot, although that slot consumes no value bytes.

### Hand-pack the levels

Repetition maximum 1 needs one bit. Pad `[0,0,0,0,1]` to one group:

```text
rep values:  0,0,0,0,1,0,0,0
packed byte: 10
hybrid:      03 10
```

Definition maximum 3 needs two bits. Pad `[0,1,2,3,3]`:

```text
def values:   0,1,2,3,3,0,0,0
packed bytes: E4 03
hybrid:       03 E4 03
```

The dense PLAIN `BYTE_ARRAY` values are:

```text
"A": 01 00 00 00 41
"B": 01 00 00 00 42
```

A V1 uncompressed body is:

```text
02 00 00 00  03 10
03 00 00 00  03 E4 03
01 00 00 00 41  01 00 00 00 42
```

Its length is:

```text
(4 + 2) + (4 + 3) + 10 = 23 bytes
```

A V2 uncompressed body omits both four-byte prefixes:

```text
03 10  03 E4 03  01 00 00 00 41 01 00 00 00 42
```

The V2 header supplies repetition length 2, definition length 3,
`num_values = 5`, and `num_rows = 4`. The value region is still only 10 bytes.

## Repetition levels and page boundaries

Every repetition-0 position begins a top-level row. In the example:

```text
rep = [0,0,0,0,1]
       ^ ^ ^ ^
       four row starts
```

A repeated row may span a **Data Page V1** boundary when no OffsetIndex is
present. The first raw position in the column chunk must start with repetition
0, but a later unindexed V1 page may start with repetition 1 because it
continues a row begun on the preceding page. Therefore page-local
`count(rep == 0)` is not always the complete V1 record count without boundary
context.

**Data Page V2 is stricter:** every V2 page must begin at a row boundary, so
its first repetition level is 0 and a repeated row cannot continue from the
preceding V2 page. Its header also carries `num_rows`. A V1 column with an
OffsetIndex has the same row-boundary requirement because each
`PageLocation.first_row_index` relies on row-aligned pages.

**Parquet format rule:** `num_values` is the raw level-entry count, not
necessarily row count or dense value count.

## Hardwood's layer model

Raw Dremel positions are useful for decoding but awkward for a columnar API.
Hardwood exposes real-items-only arrays with schema layers.

**Hardwood choice:** along the root-to-leaf path:

- an optional user-authored group contributes a `STRUCT` layer with validity;
- a `LIST`- or `MAP`-annotated group contributes a `REPEATED` layer with
  validity and sentinel-suffixed offsets;
- required groups and synthetic LIST/MAP scaffolding contribute no layer.

For the four-state list, Hardwood's public view is:

```text
layer kind:      REPEATED
list offsets:    [0, 0, 0, 1, 3]
list validity:   [null, present, present, present]

real leaf slots: [null, "A", "B"]
leaf validity:   [null, present, present]
value count:     3
```

Interpret each row:

- row 0: zero-width offset span and list validity null → null list;
- row 1: zero-width span and list validity present → empty list;
- row 2: span `[0,1)` and present list → one element; leaf validity says null;
- row 3: span `[1,3)` → `"A"` and `"B"`.

The trailing offset is a sentinel equal to the next inner item count, here the
leaf value count. For nested lists, outer offsets point into inner-list items,
and inner offsets point into leaf items.

This produces one more crucial count distinction:

```text
raw Page.size()              = 5
ColumnReader.getValueCount() = 3
dense on-disk value count    = 2
```

**Hardwood choice:** phantom positions for null or empty repeated parents are
removed from the public leaf axis. A null element remains because it is a real
slot. Validity uses set-bit-means-present polarity, and an omitted internal
bitmap means all items at that scope are present.

## Format rules and Hardwood choices

**Parquet format rule**

- Definition levels encode how much of the schema path is defined.
- Repetition levels encode continuation at repeated depths.
- A null/empty container still needs a raw position to preserve row alignment.
- Only maximum-definition positions consume dense leaf values.
- An unindexed Data Page V1 can begin with continuation of a repeated record
  from a previous page. Data Page V2, and any page covered by an OffsetIndex,
  must begin at a row boundary.

**Hardwood choice**

- Decoded pages initially retain raw positions and their level arrays.
- `NestedColumnWorker` assembles page fragments into record-aligned batches.
- `NestedLevelComputer` derives STRUCT validity, REPEATED validity/offsets,
  leaf validity, and a real-to-raw compaction map.
- `ColumnReader` exposes real items only; STRUCT layers have no offsets and
  REPEATED layers do.

## Current Hardwood code tour

1. [`NestedSchemaTest`](../../core/src/test/java/dev/hardwood/NestedSchemaTest.java)
   verifies the standard list path reaches max definition 3 and max repetition
   1 for an optional element.
2. [`PageDecoder`](../../core/src/main/java/dev/hardwood/internal/reader/PageDecoder.java)
   decodes one repetition and definition entry per raw `numValues`, counts
   maximum definitions, and produces a typed `Page`.
3. [`NestedColumnWorker`](../../core/src/main/java/dev/hardwood/internal/reader/NestedColumnWorker.java)
   treats repetition 0 as a record start and preserves continuation across
   pages while assembling batches.
4. [`NestedLevelComputer`](../../core/src/main/java/dev/hardwood/internal/reader/NestedLevelComputer.java)
   computes layer descriptors and the real-items view. In `computeRealView`,
   inspect the separate conditions for a layer item, a leaf slot, and a present
   leaf value.
5. [`LayerKind`](../../core/src/main/java/dev/hardwood/reader/LayerKind.java)
   defines the two public layer kinds.
6. [`ColumnReader`](../../core/src/main/java/dev/hardwood/reader/ColumnReader.java)
   documents real-item counts, sentinel offsets, and validity. Compare
   `getRecordCount`, `getValueCount`, `getLayerOffsets`, and
   `getLeafValidity`.
7. [`ColumnReaderLayerModelTest`](../../core/src/test/java/dev/hardwood/ColumnReaderLayerModelTest.java)
   pins null-versus-empty offsets and list-of-string access.

## Guided lab

### 1. Inspect null and empty lists

The checked-in
[`list_basic_test.parquet`](../../core/src/test/resources/list_basic_test.parquet)
contains:

```text
tags = [["a","b","c"], [], null, ["single"]]
```

Run:

```shell
hardwood schema -f core/src/test/resources/list_basic_test.parquet
hardwood inspect columns -f core/src/test/resources/list_basic_test.parquet \
  --column tags.list.element
hardwood print -f core/src/test/resources/list_basic_test.parquet
```

From the schema, derive `maxDefinitionLevel` and `maxRepetitionLevel` before
looking at source. In the column inspection, compare level counts with the four
top-level rows.

### 2. Inspect a null element

The checked-in
[`uuid_list_test.parquet`](../../core/src/test/resources/uuid_list_test.parquet)
contains a two-element list, a one-slot list whose element is null, and an
empty list:

```shell
hardwood print -f core/src/test/resources/uuid_list_test.parquet
hardwood inspect columns -f core/src/test/resources/uuid_list_test.parquet \
  --column session_ids.list.element
```

Explain why the null-element row has a one-element offset span while the empty
row has a zero-element span.

### 3. Run focused schema and layer tests

```shell
timeout 180s ./mvnw -pl core \
  -Dtest=NestedSchemaTest#testListSchema test

timeout 180s ./mvnw -pl core \
  -Dtest=ColumnReaderLayerModelTest#listOfStringCrossProductOfOffsets test
```

Read the focused methods before running them. Match each assertion to one of:
schema-derived maximum levels, list validity, list offsets, binary offsets, or
leaf values.

### 4. Trace compaction

Use the hand-worked arrays:

```text
rep = [0,0,0,0,1]
def = [0,1,2,3,3]
```

Trace
[`NestedLevelComputer.computeRealView`](../../core/src/main/java/dev/hardwood/internal/reader/NestedLevelComputer.java).
For each raw index, record whether it creates:

- a top-level list slot;
- a real leaf slot; and
- a present leaf value.

Your final offsets must be `[0,0,0,1,3]`, with three real leaf slots and two
present values.

## Common misconceptions

- **“Null list and empty list have the same levels.”** Both have no values, but
  their definitions differ.
- **“A null element is the same as a null list.”** A null element creates a
  real list slot; a null list does not.
- **“One row always produces one level entry.”** Repeated rows can produce many
  entries; null/empty rows still produce one placeholder.
- **“`num_values` is the number of leaf values exposed by Hardwood.”** It is
  the raw position count. Hardwood compacts phantom positions.
- **“Repetition 1 means the second row.”** It means continuation at repeated
  depth 1; repetition 0 starts the next row.
- **“Every page starts at a row boundary.”** Data Page V2 and indexed pages do,
  but a later unindexed Data Page V1 may continue a repeated row.
- **“STRUCT layers need offsets.”** They do not expand or contract the item
  stream; they carry validity only.

## Recap and debugging checklist

- [ ] Draw the exact root-to-leaf schema path.
- [ ] Increment definition for optional and repeated path nodes.
- [ ] Increment repetition for repeated path nodes.
- [ ] Assign a semantic meaning to every definition level.
- [ ] Count raw positions, real leaf slots, dense values, and rows separately.
- [ ] Treat repetition 0 as a top-level record start.
- [ ] Preserve continuation when an unindexed V1 page begins above repetition 0.
- [ ] Require a V2 page to begin at repetition 0.
- [ ] Require pages described by an OffsetIndex to begin at repetition 0.
- [ ] Use layer validity to distinguish null from empty zero-span containers.
- [ ] Keep null elements as real slots with cleared leaf validity.

## Quiz

1. For the standard optional-list/optional-element schema, what are the leaf's
   maximum definition and repetition levels?
2. Write `(rep, def, value?)` entries for
   `null`, `[]`, `[null]`, and `["A","B"]`.
3. For that four-row example, give row count, raw position count, real leaf-slot
   count, and dense value count.
4. What bit widths are used for its repetition and definition streams? Verify
   the packed hybrid bytes `03 10` and `03 E4 03`.
5. Derive Hardwood's list offsets, list validity, and leaf validity for the
   example.
6. A later Data Page V1 starts with repetition level 1. Is it necessarily
   corrupt, and what context is required? How do Data Page V2 and the presence
   of an OffsetIndex change the answer?
7. An optional `profile` struct contains an optional `tags` list with optional
   elements. Which Hardwood layers lie between root and the element leaf?
8. In `computeRealView`, which raw definitions from `[0,1,2,3,3]` create real
   leaf slots, and which consume dense values?

Compare your work with the
[answer key](../answers/10-nested-data.md).
