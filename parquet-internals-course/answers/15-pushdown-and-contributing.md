# Answers — Chapter 15: Skipping Work and Contributing Safely

[Return to the lesson](../chapters/15-pushdown-and-contributing.md)

## 1. Region decisions and exact filtering

- `CANNOT_MATCH` is a proof that no row in the region can make the predicate
  true. The region is omitted, so no exact filtering is needed.
- `MIGHT_MATCH` means metadata cannot decide. The region survives and its
  records require exact filtering unless a later proof narrows the state.
- `ALWAYS_MATCHES` proves every row in the region satisfies the predicate.
  The region is read, but exact predicate evaluation can be skipped.

Thus only `MIGHT_MATCH` requires residual evaluation. “Might” deliberately
includes both true matches and false positives.

## 2. Bloom miss versus hit

A correctly implemented Parquet bloom filter has no false negatives for
values represented in it. If the probed equality value is definitely absent,
that value cannot occur in the column chunk, so the equality cannot match.

A hit is only “possibly present” because bloom filters permit false positives.
It does not prove that the value occurs, much less that every non-null row
equals it and that nulls cannot invalidate the predicate. Therefore it cannot
produce `ALWAYS_MATCHES`.

## 3. Range intersection and union

Given:

```text
A = [0,100) [300,500)
B = [50,350)
```

Intersection:

```text
A AND B = [50,100) [300,350)
```

Union initially combines all three intervals. They overlap in a chain:
`[0,100)` overlaps `[50,350)`, which overlaps `[300,500)`, so:

```text
A OR B = [0,500)
```

## 4. Page-relative mask

The row-range/page overlap is:

```text
[420,480) intersect [400,450) = [420,450)
```

Subtract page start 400 from both endpoints:

```text
PageRowMask = [20,50)
```

The final 30 top-level records of that page are kept.

## 5. Why both page indexes are needed

`ColumnIndex` says which page-statistics entries can or cannot satisfy the
predicate, but it does not provide the complete row and byte location needed
for that page.

`OffsetIndex` maps the same page ordinal to `firstRowIndex`, file offset, and
compressed page size. Hardwood needs matching ordinals to turn a keep/drop
decision into `RowRanges` and later ranged I/O. It verifies equal page counts;
missing either side conservatively yields all rows.

## 6. Unsafe inline drop under `OR`

For:

```text
x > 10 OR y = 2
```

an impossible `x > 10` branch does not make the full predicate impossible.
Rows can still satisfy `y = 2`. SQL evaluation itself does not lose such a row:
`UNKNOWN OR TRUE` is `TRUE`. The danger is that replacing a present `x` with a
null placeholder fabricates the projected value for a row admitted by `y`.
Correct row counts can therefore hide corrupt output values.

Inline placeholder extraction therefore uses only leaves necessary to the
whole predicate—leaves reached through `AND`, not arbitrary leaves under
`OR`. A regression must compare projected values as well as selected rows.

## 7. Testing “correct rows but scanned too much”

Add an observation of the work boundary, not another value assertion. For a
sequential trailing-scan bug, record `RowGroupScannedEvent.pageCount` and
compare a bounded range with an all-pages baseline, as
`SequentialFetchPlanEarlyExitTest` does.

If the claim is specifically excess ranged bytes, wrap the input in
`CountingInputFile` and assert filtered bytes are lower. Place the first test
at fetch-plan/I/O integration, where header scans or range requests are
observable. A row-reader result alone cannot prove the optimization occurred.

## 8. Minimum safe fixture-backed workflow

1. Confirm or create the relevant GitHub issue before changing code. Inspect
   the repository's label set and apply the relevant labels (`bug` for a
   defect, plus affected-area labels); use `good first issue` and `help wanted`
   together only for genuinely suitable newcomer work.
2. Reproduce the first broken invariant with a focused test and run it to see
   it fail.
3. If bytes/metadata are required, add a small deterministic recipe to
   `tools/simple-datagen.py`, using `tools/parquet_annotators.py` for metadata
   PyArrow cannot emit.
4. Generate with the repository `.docker-venv`; `pyarrow==24.0.0` and
   `thriftpy2==0.6.0` are exact, load-bearing pins because they produce
   checked-in bytes.
5. Implement the fix at the layer that authorized or violated the invariant.
6. Add the smallest public/end-to-end regression needed for semantics and an
   I/O/event assertion when skipped work is part of the claim.
7. Update `docs/content/` for public API/behavior surfaces and an `_designs/`
   end-state document plus roadmap status for architectural work.
8. Run focused tests, formatting/source processing, then
   `timeout 180s ./mvnw clean verify`.
9. Commit under the issue-prefixed message convention with a body explaining
   why the change is needed, and review the generated fixture diff rather than
   treating it as opaque output.

The test-first step prevents a green suite from being mistaken for evidence
that the reported defect was ever reproduced.
