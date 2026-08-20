# Answers — Chapter 10: Nested Data and Dremel Levels

Return to the [lesson](../chapters/10-nested-data.md).

## 1. Maximum levels

For:

```text
optional group tags (LIST) {
  repeated group list {
    optional binary element;
  }
}
```

definition rises once for each optional or repeated node:

```text
optional tags + repeated list + optional element = 3
```

Only the repeated `list` node raises repetition:

```text
max definition = 3
max repetition = 1
```

## 2. Level entries

The sequence is:

```text
null       -> (0, 0, no value)
[]         -> (0, 1, no value)
[null]     -> (0, 2, no value)
["A","B"]  -> (0, 3, "A"), (1, 3, "B")
```

The first entry of every top-level row has repetition 0. `"B"` continues row
3's list, so it has repetition 1. Only definition 3 reaches the leaf maximum
and consumes a dense value.

## 3. Four distinct counts

```text
row count = 4
```

The first three rows emit one raw placeholder each and the last emits two:

```text
raw positions = 1 + 1 + 1 + 2 = 5
```

`[null]` has one real element slot and `["A","B"]` has two:

```text
real leaf slots = 0 + 0 + 1 + 2 = 3
```

Only `"A"` and `"B"` are present:

```text
dense values = 2
```

## 4. Bit widths and packed bytes

Maximum repetition 1 requires one bit. The padded group is:

```text
0,0,0,0,1,0,0,0
```

Only bit 4 is set, producing payload `10`. One packed group has header:

```text
(1 << 1) | 1 = 03
```

So repetition bytes are **`03 10`**.

Maximum definition 3 requires two bits. Pack:

```text
0,1,2,3 -> 0 + (1<<2) + (2<<4) + (3<<6)
          = 0 + 4 + 32 + 192
          = 228 = E4

3,0,0,0 -> 3 = 03
```

With the same one-group header, definition bytes are **`03 E4 03`**.

## 5. Hardwood layer view

There is one `REPEATED` layer. Cumulative real element counts after each row
give sentinel-suffixed offsets:

```text
start          = 0
after null     = 0
after empty    = 0
after [null]   = 1
after ["A","B"]= 3

offsets = [0, 0, 0, 1, 3]
```

The list itself is null only in row 0:

```text
list validity = [null, present, present, present]
```

The three real leaf slots are the null element, `"A"`, and `"B"`:

```text
leaf validity = [null, present, present]
```

Rows 0 and 1 both have zero spans, so list validity is necessary to distinguish
null from empty.

## 6. Page starting at repetition 1

It is not necessarily corrupt. A repeated top-level record can begin on the
previous page and continue on this one. The reader needs preceding
column-chunk/page context to know whether an open record exists.

The first raw position of the entire column chunk must start a record with
repetition 0. A repetition-1 first position is invalid only if there is no
preceding repeated record to continue.

## 7. Hardwood layers under `profile.tags`

The optional user-authored `profile` group contributes:

```text
STRUCT
```

The LIST-annotated `tags` group contributes:

```text
REPEATED
```

The synthetic repeated list wrapper contributes no separate layer, and the
optional element is the leaf rather than an intermediate layer. Outermost to
innermost:

```text
[STRUCT, REPEATED]
```

The STRUCT carries profile validity. The REPEATED layer carries list validity
and offsets.

## 8. Real slots and dense consumption

For definitions `[0,1,2,3,3]`:

- definition 0 is a null list: no real leaf slot;
- definition 1 is an empty list: no real leaf slot;
- definition 2 reaches the list item but not the optional leaf: create one
  real null leaf slot, consume no dense value;
- each definition 3 creates a real present leaf slot and consumes one dense
  value.

Therefore raw indexes 2, 3, and 4 create real leaf slots. Only indexes 3 and 4
consume dense values. This is why compaction maps five raw positions to three
public leaf positions backed by two encoded values.
