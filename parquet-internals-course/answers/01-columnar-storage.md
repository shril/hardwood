# Answers — Chapter 1: From SQL Tables to Columnar Storage

Return to
[`../chapters/01-columnar-storage.md`](../chapters/01-columnar-storage.md).

## 1. Projection and filtering

**Projection** chooses which columns a query returns. **Filtering** uses a
predicate to choose which rows survive.

They can interact: a filter's input column may have to be read even when it is
not part of the projection.

## 2. Fixed-width sizing

Each complete row occupies:

```text
8 + 4 + 16 = 28 bytes
```

For 1,000 rows, total value bytes are:

```text
1,000 × 28 = 28,000 bytes
```

The projection of the 4-byte column needs:

```text
1,000 × 4 = 4,000 bytes
```

It avoids:

```text
28,000 - 4,000 = 24,000 bytes
```

As a fraction:

```text
24,000 / 28,000 = 6/7 ≈ 0.8571
```

So it avoids about **85.7%** of the value bytes. This calculation deliberately
excludes metadata, headers, null structure, and compression.

## 3. Projection plus predicate

A straightforward correct reader must decode both `name` and `age`: `age` is
needed to evaluate the predicate, and `name` is needed to construct the output.
Only `name` appears in the projected result.

Metadata might prove that an entire region cannot match and thereby avoid both
columns for that region, but that is an additional optimization rather than a
change to the query's data dependencies.

## 4. A group and its leaves

There are **two** stored leaf columns:

```text
address.city
address.zip
```

`address` is a group node. It contributes structure and a path component but
does not have its own primitive value stream.

## 5. Correct output is not I/O evidence

The reader could fetch every column, discard all but the projected one, and
still return exactly the expected values. Output correctness therefore proves
decoding semantics, not physical I/O avoidance.

The stronger boundary is
[`InputFile.readRange(long, int)`](../../core/src/main/java/dev/hardwood/InputFile.java).
A counting or recording `InputFile` can show which offsets and lengths the
reader actually requested.

## 6. Conceptual call trace

[`ColumnProjection`](../../core/src/main/java/dev/hardwood/schema/ColumnProjection.java)
holds the requested string `address.city`.

[`ProjectedSchema`](../../core/src/main/java/dev/hardwood/internal/schema/ProjectedSchema.java)
resolves requested names against the full schema and produces the concrete
selected leaves.

[`InputFile`](../../core/src/main/java/dev/hardwood/InputFile.java) ultimately
serves the planned byte ranges through `readRange(long, int)`.

There are planning and worker classes between those points, but those are the
three boundaries requested by the question.

## 7. Format rule or Hardwood choice

1. **Format rule:** one column chunk holds one leaf's data for one row group.
   This is part of Parquet's physical hierarchy.
2. **Hardwood choice:** exposing decoded `INT64` values as `long[]` belongs to
   Hardwood's Java API.
3. **Format rule:** primitive schema leaves carry stored values, while groups
   describe structure.
4. **Hardwood choice:** the `readRange(long, int)` Java method is Hardwood's
   storage abstraction. Parquet requires readers to locate data but does not
   prescribe that interface.
