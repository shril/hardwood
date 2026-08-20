# Chapter 15 — Skipping Work and Contributing Safely

**Study time:** 45–75 minutes

A correct filter can be evaluated after every row is decoded. Predicate
pushdown keeps that exact semantics while proving that some bytes, pages, or
row groups cannot contribute. This final chapter joins the pruning layers,
row masks, SQL null logic, and the contribution workflow needed to change
them without introducing silent data loss.

## Objectives

By the end of this lesson, you should be able to:

1. distinguish row-group pruning, page-index pruning, inline-page fallback,
   and exact record filtering;
2. compose `FilterDecision` and `RowRanges` conservatively for `AND` and `OR`;
3. trace row-group-relative ranges into page-relative `PageRowMask`s and batch
   assembly;
4. explain why SQL null/unknown semantics constrain metadata shortcuts;
5. choose the earliest useful failing-test boundary for a reader defect;
6. design and verify a fixture-backed contribution under Hardwood's issue,
   documentation, generator, and build rules.

## Prerequisite recap

- Footer `ColumnMetaData.statistics` summarizes an entire column chunk.
- Bloom filters can prove that a probed value is definitely absent, but a
  positive result may be a false positive.
- A fully covering dictionary is an exact set of non-null values for eligible
  dictionary-encoded chunks.
- A `ColumnIndex` stores per-page min/max/null information; a matching
  `OffsetIndex` maps those page entries to byte ranges and first row indices.
- Data page headers may carry inline statistics even when no page index exists.
- A filter over decoded rows remains the semantic authority whenever metadata
  cannot prove a shortcut.

## Central mental model: a ladder of conservative proofs

Pushdown is a sequence of gates:

```text
public FilterPredicate
        |
        v
resolved, typed predicate
        |
        v
row-group decision
  statistics + bloom + eligible dictionary
  CANNOT_MATCH -> omit work item
  ALWAYS_MATCHES -> keep, may skip exact evaluation
  MIGHT_MATCH -> keep and continue
        |
        v
page-index decision
  ColumnIndex statistics + OffsetIndex locations
  -> RowRanges of rows that might match
        |
        v
fetch plans
  omit page body / narrow ranged read
  -> PageRowMask for boundary/overlap pages
        |
        v
inline-stat fallback for sequential pages
  optional AND-necessary column page proved impossible
  -> null placeholder, no body decode
        |
        v
exact filtering of surviving decoded records
  SQL null logic determines result
```

Every downward transition may keep false positives. None may create a false
negative. “Might match” is therefore a success state, not a failure of the
optimizer.

The safest debugging question is:

> What proof authorized this work to be skipped, and where is that proof
> represented?

## Worked example: two columns and overlapping page ranges

Consider 1,000 rows with:

```sql
WHERE price > 100 AND region = 'EU'
```

### Row-group gate

Suppose row-group statistics say:

```text
price:  min=20, max=500, nullCount=5
region: min="APAC", max="US", nullCount=0
```

Neither range proves the conjunction impossible. A region bloom filter also
reports “possibly present.” The decision is `MIGHT_MATCH`; the row group
survives.

If an eligible complete region dictionary lacked `"EU"`, that exact absence
would change the decision to `CANNOT_MATCH`, and no page index or data bytes
would be needed.

### Page-index gate

Assume per-page statistics produce:

```text
price > 100 might match rows:   [200, 600) [800, 1000)
region = EU might match rows:   [400, 900)
```

`AND` intersects ranges:

```text
[400, 600) [800, 900)
```

These ranges are an over-approximation. Page statistics say only that some
value in each contributing page might satisfy the leaf; they do not establish
row-by-row correlation between `price` and `region`.

For:

```sql
price > 100 OR region = 'EU'
```

the evaluator unions them:

```text
[200, 1000)
```

Using intersection for `OR` would silently lose rows.

### Ranges become masks

Suppose another projected column `order_id` has a page covering rows
`[350, 550)`. Intersecting that page with the `AND` ranges gives
`[400, 550)`. `RowRanges.maskForPage` subtracts page start:

```text
PageRowMask [50, 200)
```

The fetch plan still needs this page because it overlaps. The column worker
decodes it and copies only those top-level records. A page covering
`[600, 800)` gets a null mask and can be omitted.

The same row-group `RowRanges` is translated independently against every
projected column's page boundaries. That is how columns with different page
layouts remain aligned.

### Exact residual and nulls

For a row with `price = NULL` and `region = 'EU'`:

```text
price > 100       -> UNKNOWN
region = 'EU'     -> TRUE
UNKNOWN AND TRUE  -> UNKNOWN
WHERE keeps only TRUE -> row does not match
```

Metadata can keep the row/page conservatively, but exact filtering must reject
it. A shortcut may bypass exact evaluation only when it proves
`ALWAYS_MATCHES`, including the required null conditions.

## The four execution layers

### 1. Row-group decisions

`RowGroupFilterEvaluator` returns:

- `CANNOT_MATCH`: safe to remove the row group;
- `MIGHT_MATCH`: retain and evaluate further;
- `ALWAYS_MATCHES`: all rows satisfy the predicate, so exact evaluation can
  be skipped for a homogeneous batch.

Statistics, bloom filters, and eligible dictionaries are independent evidence
sources. For equality and membership, definitely absent bloom/dictionary
evidence can drop a group even when the value lies inside min/max.

Missing, malformedly unavailable, or inapplicable evidence normally falls
back to `MIGHT_MATCH`. It must not be interpreted as absence.

`AND` and `OR` combine the three decisions logically:

```text
AND: one CANNOT_MATCH drops; all ALWAYS_MATCHES proves always
OR:  all CANNOT_MATCH drops; one ALWAYS_MATCHES proves always
```

### 2. Page-index ranges

`PageFilterEvaluator` needs both `ColumnIndex` and `OffsetIndex` for a leaf:

- `ColumnIndex` answers whether each page can be dropped from its statistics;
- `OffsetIndex` turns page index `i` into row interval
  `[firstRowIndex[i], firstRowIndex[i+1])`.

It verifies both indexes describe the same page count before walking them.
Missing either index yields all rows. For predicates:

```text
AND -> RowRanges.intersect
OR  -> RowRanges.union
```

An all-null page can be omitted for ordinary comparisons and `IS NOT NULL`.
`IS NULL` can omit a page only when a known null count is zero.

### 3. Fetch-plan masks

`RowRanges` contains sorted, non-overlapping, half-open row-group-relative
intervals. It supports:

- `ALL`/`all(rowCount)` for conservative no-pruning state;
- `fromPages` to merge adjacent kept page intervals;
- `intersect` and `union`;
- `endRow` for sequential early exit;
- `maskForPage` to produce page-relative intervals.

Mask meanings are precise:

```text
PageRowMask.ALL -> keep every top-level record
nontrivial mask -> keep listed page-relative intervals
null            -> keep no records; omit page
```

For flat columns record offsets equal value positions. For nested columns they
count repetition-level-zero record starts. Hardwood applies a row-group-wide
capability gate: if every projected column cannot honor the same selection,
the plans use all rows and exact filtering preserves correctness.

### 4. Inline-stat sequential fallback

Without a `ColumnIndex`, a sequential plan sees each page header while
scanning. `PageDropPredicates` extracts only leaves that are individually
necessary to the full predicate: it descends through `AND` and does not make
unsafe claims through `OR`.

If inline statistics prove such a page cannot satisfy its necessary leaf,
Hardwood can avoid the real page body for an optional column and emit a typed
all-null placeholder with the same record count. Exact filtering then sees:

```text
null <op> constant -> UNKNOWN -> not selected by WHERE
```

Sibling columns remain row-aligned. Required columns cannot use this trick
because synthesizing null violates their schema, so they decode normally.

## SQL null logic is part of correctness

Do not conflate two different three-valued systems:

1. `FilterDecision` describes what metadata proves about a whole region:
   cannot, might, or always match.
2. SQL evaluates expressions per row as `TRUE`, `FALSE`, or `UNKNOWN`.

They interact, but are not the same enum or algorithm.

Useful SQL identities:

```text
NULL = 5            -> UNKNOWN
NULL < 5            -> UNKNOWN
NULL IN (1, 2)      -> UNKNOWN
NULL IS NULL        -> TRUE
NULL IS NOT NULL    -> FALSE
TRUE AND UNKNOWN    -> UNKNOWN
FALSE AND UNKNOWN   -> FALSE
TRUE OR UNKNOWN     -> TRUE
FALSE OR UNKNOWN    -> UNKNOWN
WHERE retains only TRUE
```

Metadata statistics are also subtle for nested columns: leaf null counts are
not always top-level row null counts. The row-group evaluator therefore
declines some tempting `ALWAYS_MATCHES` conclusions, such as treating
`nullCount == numRows` as universal `IS NULL` for every nested shape.

## Parquet format rules and Hardwood choices

### Format rules

- Row-group/column-chunk statistics, bloom filters, dictionaries, Column
  Indexes, Offset Indexes, and inline page statistics are optional metadata
  structures with defined representations.
- Bloom filters can prove absence but not presence.
- Column Index entries correspond to data pages; Offset Index locations carry
  row starts and byte locations.
- Null counts and min/max have physical/logical interpretation rules that a
  reader must honor.
- Parquet defines no SQL `WHERE` API or required pushdown strategy.

### Hardwood choices

- Public predicates are resolved to typed, column-indexed internal predicates
  before evaluation.
- Row-group metadata produces a tri-state `FilterDecision`.
- Statistics, bloom, and eligible dictionary absence are combined as
  independent proofs.
- Page-index results are represented as `RowRanges`, composed recursively for
  `AND`/`OR`, then translated into each column's page layout.
- A row-group-wide mask-capability gate protects cross-column alignment.
- Sequential inline fallback considers only AND-necessary leaves and uses
  optional-column null placeholders.
- Exact filtering remains responsible for surviving `MIGHT_MATCH` rows.
- A reader option can disable metadata-derived filtering shortcuts to provide
  a full-scan correctness fallback for files with untrustworthy metadata.

## Current code tour

Follow the decision from public API to residual evaluation:

1. [`FilterPredicate`](../../core/src/main/java/dev/hardwood/reader/FilterPredicate.java)
   defines the public expression tree.
2. [`FilterPredicateResolver`](../../core/src/main/java/dev/hardwood/internal/predicate/FilterPredicateResolver.java)
   resolves names and literal types into `ResolvedPredicate`.
3. [`ResolvedPredicate`](../../core/src/main/java/dev/hardwood/internal/predicate/ResolvedPredicate.java)
   is the typed internal tree.
4. [`FilterDecision`](../../core/src/main/java/dev/hardwood/internal/predicate/FilterDecision.java)
   implements tri-state `and`/`or`.
5. [`RowGroupFilterEvaluator`](../../core/src/main/java/dev/hardwood/internal/predicate/RowGroupFilterEvaluator.java)
   combines statistics, bloom, dictionary, null, and geospatial evidence.
6. [`StatisticsFilterSupport`](../../core/src/main/java/dev/hardwood/internal/predicate/StatisticsFilterSupport.java)
   contains min/max leaf reasoning.
7. [`PageFilterEvaluator`](../../core/src/main/java/dev/hardwood/internal/predicate/PageFilterEvaluator.java)
   parses paired indexes and constructs recursive ranges.
8. [`RowGroupIndexBuffers`](../../core/src/main/java/dev/hardwood/internal/reader/RowGroupIndexBuffers.java)
   fetches/slices row-group index regions.
9. [`RowRanges`](../../core/src/main/java/dev/hardwood/internal/reader/RowRanges.java)
   implements row-group-relative interval algebra.
10. [`PageRowMask`](../../core/src/main/java/dev/hardwood/internal/reader/PageRowMask.java)
    defines page-relative top-level record selections.
11. [`PageDropPredicates`](../../core/src/main/java/dev/hardwood/internal/predicate/PageDropPredicates.java)
    extracts safe inline-stat leaves.
12. [`SequentialFetchPlan`](../../core/src/main/java/dev/hardwood/internal/reader/SequentialFetchPlan.java)
    applies inline drops, placeholders, masks, and early exit.
13. [`PageDecoder`](../../core/src/main/java/dev/hardwood/internal/reader/PageDecoder.java)
    constructs placeholder pages and rejects placeholders for required leaves.
14. [`FlatRowReader`](../../core/src/main/java/dev/hardwood/internal/reader/FlatRowReader.java)
    shows drain-side exact filter masks; other paths use record-level
    evaluation.
15. [`PARSING_PIPELINE_V2`](../../_designs/PARSING_PIPELINE_V2.md) records the
    intended architecture and rationale. Current source and tests override
    stale operational details in that completed design.

Tests:

- [`RowGroupDecideTest`](../../core/src/test/java/dev/hardwood/internal/predicate/RowGroupDecideTest.java)
  pins tri-state combinations.
- [`PageFilterEvaluatorTest`](../../core/src/test/java/dev/hardwood/internal/predicate/PageFilterEvaluatorTest.java)
  pins page range algebra and index behavior.
- [`PageRangeIoTest`](../../core/src/test/java/dev/hardwood/internal/reader/PageRangeIoTest.java)
  proves fewer bytes, not just correct output.
- [`NestedV2NoIndexMaskingTest`](../../core/src/test/java/dev/hardwood/NestedV2NoIndexMaskingTest.java)
  exercises sequential nested masking.
- [`NestedV1NoIndexFallbackTest`](../../core/src/test/java/dev/hardwood/NestedV1NoIndexFallbackTest.java)
  pins the conservative capability fallback.
- [`MetadataFilteringOptionTest`](../../core/src/test/java/dev/hardwood/MetadataFilteringOptionTest.java)
  compares metadata shortcuts with full-scan evaluation.

## Failure-first test placement

For a bug report, first reproduce the broken invariant at the narrowest layer
that can express both the input and expected behavior:

| Broken invariant | First useful test boundary | Add end-to-end coverage when |
|---|---|---|
| malformed Thrift count accepted | internal metadata reader test or generated corrupt fixture | file attribution/public path matters |
| min/max predicate result wrong | `StatisticsFilterSupport` / evaluator unit test | physical/logical conversion caused the case |
| `AND`/`OR` ranges wrong | `PageFilterEvaluatorTest` / `RowRangesTest` | actual I/O or cross-column alignment regressed |
| bytes not skipped | `CountingInputFile` integration test | always: value correctness alone cannot prove I/O |
| trailing headers still scanned | JFR page-count test | returned rows are already correct |
| page decode wrong | focused decoder/encoding test | assembly or public logical values also matter |
| batch alignment/ownership wrong | worker or array-identity regression | public contract is the failure |

The failure should fail before the fix. A broad end-to-end test that says
“wrong rows” is valuable, but it may not identify whether the proof, range,
fetch, decode, or residual layer broke. Prefer one focused invariant test plus
the smallest public regression that protects the reported behavior.

## Fixture generator constraints

Real binary edge cases belong in reproducible fixtures, not opaque hand-edited
blobs.

- Extend [`tools/simple-datagen.py`](../../tools/simple-datagen.py).
- Use helpers in
  [`tools/parquet_annotators.py`](../../tools/parquet_annotators.py) when
  PyArrow cannot emit the required annotation/corruption.
- Run Python from `.docker-venv` with Python 3.10–3.14:

  ```shell
  source .docker-venv/bin/activate
  python tools/simple-datagen.py
  ```

- `pyarrow==24.0.0` and `thriftpy2==0.6.0` in
  [`requirements.txt`](../../requirements.txt) are load-bearing exact pins.
  They directly produce checked-in bytes. An upgrade requires regenerating
  and reviewing affected fixtures, not merely changing a dependency line.
- Keep fixture data small and encode the intended boundary visibly in the
  generator: row-group count, page size, null pattern, sort order, codec,
  index presence, and metadata mutation.
- A fixture test should state why ordinary writer output cannot cover the case
  and what byte/metadata property the annotator creates.

## Guided lab and contribution capstone

Run from the repository root.

### 1. Hand-compose row-group decisions

Read `FilterDecision.and` and `or`, then complete both 3×3 truth tables for:

```text
CANNOT_MATCH, MIGHT_MATCH, ALWAYS_MATCHES
```

Check your table:

```shell
timeout 180s ./mvnw -pl core -Dtest=RowGroupDecideTest,FilterDecisionTest test
```

Explain one case where bloom evidence changes `MIGHT_MATCH` to
`CANNOT_MATCH`, and why it can never change it to `ALWAYS_MATCHES`.

### 2. Trace ranges into a mask

On paper:

```text
A ranges = [0,100) [300,500)
B ranges = [50,350)
```

Compute `A AND B` and `A OR B`. For a page `[275,375)`, convert each result
to page-relative mask intervals.

Then run the focused interval/index tests:

```shell
timeout 180s ./mvnw -pl core -Dtest=RowRangesTest,PageFilterEvaluatorTest test
```

### 3. Compare optimized and full-scan semantics

```shell
timeout 180s ./mvnw -pl core -Dtest=MetadataFilteringOptionTest,PageRangeIoTest test
```

For each suite, classify assertions as:

- semantic equivalence;
- reduced I/O;
- controlled fallback.

An optimization change is incomplete if it proves only fewer bytes without
the same rows, or only the same rows without proving the intended work was
skipped.

### 4. Inspect nested capability fallback

```shell
timeout 180s ./mvnw -pl core -Dtest=NestedV2NoIndexMaskingTest,NestedV1NoIndexFallbackTest test
```

Trace why V2 repetition-level prefixes can support record counts without value
decompression, while nested V1 causes the whole row group's masks to fall
back. State the data-loss risk of masking only the V2-capable sibling.

### 5. Design a failure-first capstone

Hypothetical report:

> For `SELECT optional_int ... WHERE optional_int > 10 OR category = "hot"`,
> a no-index file retains the `"hot"` rows but returns `NULL` for
> `optional_int` values that are actually present and at most 10.

Produce a contribution plan, not a speculative code patch:

1. **Issue/context.** Confirm an issue exists. For the course work itself,
   issue `#986` is the contribution context; a real unrelated bug needs its
   own issue. Inspect the repository label set and apply `bug` here, plus any
   affected-area labels. Use `good first issue` and `help wanted` together
   only when the work is genuinely suitable for an external newcomer.
2. **First failing test.** Place a small predicate-extraction unit test showing
   that a leaf under `OR` is not AND-necessary.
3. **Public regression.** Add a generated fixture with multiple pages,
   optional nulls, inline statistics, no Column Index, and rows matching only
   the other OR branch. Assert both selected row identities **and projected
   `optional_int` values** equal metadata-disabled results; row counts alone
   would miss the fabricated-null corruption.
4. **Work assertion.** Instrument `CountingInputFile` or an event only if the
   fix is intended to retain a safe skip; correctness is primary.
5. **Fix boundary.** Inspect `PageDropPredicates`, not page decoding or row
   assembly, because extraction authorized the unsafe placeholder.
6. **Generator.** Add the fixture recipe to `simple-datagen.py`; use pinned
   `.docker-venv` tools.
7. **Docs/design.** If public behavior/API changes, update `docs/content/`.
   If architecture changes, write/update a completed end-state design under
   `_designs/` and update roadmap status.
8. **Verification.** Run focused tests, formatting, and full verification:

   ```shell
   timeout 180s ./mvnw -pl core -Dtest=PageFilterEvaluatorTest,MetadataFilteringOptionTest test
   timeout 180s ./mvnw process-sources
   timeout 180s ./mvnw clean verify
   ```

   Add the new failure-first class to the focused command once it exists; the
   command shown is a runnable baseline over the current relevant suites.
9. **Commit context.** A real commit message begins with the relevant issue
   key and explains why the proof was unsafe. Do not claim completion from a
   green broad test if the failure-first test never failed.

Review your plan against
[`CONTRIBUTING.md`](../../CONTRIBUTING.md),
[`FORMAT_COVERAGE.md`](../../FORMAT_COVERAGE.md), and
[`ROADMAP.md`](../../ROADMAP.md).

## Common misconceptions

**“Statistics say a predicate matches, so every row matches.”**  
Min/max usually establish only possibility. `ALWAYS_MATCHES` requires a much
stronger proof, including null behavior.

**“A bloom-filter hit proves the value exists.”**  
It proves only “possibly present.” A miss can prove absence.

**“Page pruning produces exact matching rows.”**  
It produces rows belonging to pages that might match. Residual evaluation
handles within-page false positives and cross-column correlation.

**“Missing metadata means the region is empty.”**  
Missing/inapplicable metadata means no proof; keep the region.

**“Inline statistics may drop either side of an OR independently.”**  
A row may match the other side. Only leaves necessary to the full expression
can authorize the placeholder shortcut.

**“A manually hex-edited fixture is enough for a regression.”**  
Without a generator recipe and pinned producer versions, future contributors
cannot reproduce or audit it reliably.

## Recap and debugging checklist

- [ ] Was the public predicate resolved to the correct file-local columns and
      literal types?
- [ ] Which evidence produced `CANNOT_MATCH`, `MIGHT_MATCH`, or
      `ALWAYS_MATCHES`?
- [ ] Is missing evidence treated conservatively?
- [ ] Do `AND`/`OR` use the correct decision and interval operations?
- [ ] Do Column Index and Offset Index page counts agree?
- [ ] Are `RowRanges` row-group-relative, sorted, and half-open?
- [ ] Are `PageRowMask` intervals page-relative top-level records?
- [ ] Can every projected column honor the mask without losing alignment?
- [ ] Is an inline-stat leaf truly AND-necessary?
- [ ] Is a null placeholder legal for the column and rejected by exact SQL
      semantics?
- [ ] Does a failing test sit at the first broken invariant?
- [ ] Does an I/O optimization test observe bytes/pages as well as values?
- [ ] Is every binary fixture reproducible under pinned generator versions?
- [ ] Do focused tests and `timeout 180s ./mvnw clean verify` pass?

## Quiz

1. Contrast `CANNOT_MATCH`, `MIGHT_MATCH`, and `ALWAYS_MATCHES`. Which states
   still require exact record filtering?
2. Why can a bloom-filter miss drop an equality row group while a hit cannot
   mark it `ALWAYS_MATCHES`?
3. Given ranges `A=[0,100),[300,500)` and `B=[50,350)`, compute `A AND B`
   and `A OR B`.
4. A kept row range is `[420,480)` and a page covers `[400,450)`. What
   `PageRowMask` interval is produced?
5. Why does page-level evaluation need both `ColumnIndex` and `OffsetIndex`?
6. Under `x > 10 OR y = 2`, why is it unsafe to replace an `x` page with
   nulls merely because that page's `x` statistics cannot satisfy `x > 10`?
7. A filtered read returns correct rows but an optimization bug report says it
   scans all trailing pages. What test observation should be added, and at
   what layer?
8. Describe the minimum safe fixture-backed bug-fix workflow from issue
   through full verification, including the two exact-pinned Python
   dependencies.

[Answer key](../answers/15-pushdown-and-contributing.md)
