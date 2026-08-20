# Answers — Chapter 5: Schema, Physical Types, and Logical Types

Return to
[`../chapters/05-schema-and-types.md`](../chapters/05-schema-and-types.md).

## 1. Physical types

The eight types are:

1. `BOOLEAN`
2. `INT32`
3. `INT64`
4. `INT96`
5. `FLOAT`
6. `DOUBLE`
7. `BYTE_ARRAY`
8. `FIXED_LEN_BYTE_ARRAY`

The two byte-array families are `BYTE_ARRAY`, whose values have variable
length, and `FIXED_LEN_BYTE_ARRAY`, whose width comes from the schema element's
`type_length`.

## 2. Flatten the tree

Depth-first pre-order emits a group before its children:

```text
0 m       group, 2 children
1 user    OPTIONAL group, 2 children
2 id      REQUIRED INT64
3 email   OPTIONAL BYTE_ARRAY, STRING
4 score   REQUIRED DOUBLE
```

The primitive leaf paths, in schema order, are:

```text
user.id
user.email
score
```

There is no `user` column chunk because `user` is a group.

## 3. Maximum levels

No node is repeated, so every maximum repetition level is zero.

For `user.id`:

```text
OPTIONAL user  -> definition +1
REQUIRED id    -> definition +0
max definition = 1
max repetition = 0
```

For `user.email`:

```text
OPTIONAL user  -> definition +1
OPTIONAL email -> definition +1
max definition = 2
max repetition = 0
```

For `score`:

```text
REQUIRED score -> definition +0
max definition = 0
max repetition = 0
```

The two levels on `user.email` distinguish a missing `user` from a present
`user` whose `email` is null.

## 4. Physical integer and DECIMAL

Decode little-endian `39 30 00 00`:

```text
0x39 × 256^0 =  57
0x30 × 256^1 =  48 × 256 = 12,288
total                         12,345
```

With no logical annotation, the value is integer **12,345**.

For `DECIMAL(7,2)`, scale 2 means:

```text
logical value = unscaled × 10^-2
              = 12,345 × 0.01
              = 123.45
```

The physical decode is identical; only logical conversion changes.

## 5. DATE value `-1`

DATE is signed days from 1970-01-01:

```text
1970-01-01 + (-1 day) = 1969-12-31
```

The physical decoder produces Java integer `-1`.
[`LogicalTypeConverter`](../../core/src/main/java/dev/hardwood/internal/conversion/LogicalTypeConverter.java)
applies the DATE annotation and calls epoch-day conversion to produce the
calendar value.

## 6. Timestamp in microseconds

Convert the count:

```text
1,500,000 microseconds
= 1,000,000 microseconds + 500,000 microseconds
= 1 second + 0.5 second
```

Starting at the Unix epoch gives:

```text
1970-01-01T00:00:01.500Z
```

Because `isAdjustedToUTC=true`, the result is an instant on the UTC timeline,
not a local wall-clock value.

## 7. Modern versus legacy annotation

In case (a), Hardwood sees a non-null modern `DateType` first and uses it. The
effective logical type is **DATE**; legacy `UTF8` is not used. Such
corresponding annotations are inconsistent and a conforming writer must not
emit them, but Hardwood's effective-type choice gives modern metadata
precedence.

In case (b), the modern annotation is absent. Hardwood maps legacy converted
type `DATE` to a `LogicalType.DateType`, so the effective logical type is again
**DATE**.

The relevant branch is in
[`FileSchema.effectiveLogicalType`](../../core/src/main/java/dev/hardwood/schema/FileSchema.java).

## 8. Format rule or Hardwood choice

1. **Format rule:** primitive schema leaves carry Parquet physical types;
   groups carry child structure.
2. **Hardwood choice:** using dot-separated strings and a lookup map for paths
   is Hardwood's schema API and implementation.
3. **Format rule:** DATE annotates an `INT32` containing signed days since the
   Unix epoch.
4. **Hardwood choice:** Java `LocalDate` is Hardwood's domain representation.
   Parquet defines the interoperable meaning, not a Java return class.
