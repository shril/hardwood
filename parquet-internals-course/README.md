# Parquet Internals for Hardwood Contributors

This course builds a working mental model of a Parquet reader, then maps that
model onto Hardwood's implementation. It is written for a Java developer who
uses SQL or data frames but has not worked with binary file formats or storage
engines.

The goal is not to memorize the Parquet specification. By the end, you should
be able to:

- draw the important regions of a Parquet file and explain why they are there;
- follow bytes from `InputFile.readRange` to a typed primitive array;
- hand-decode small metadata, level, PLAIN, and RLE/bit-packed examples;
- explain how nulls, lists, and nested records survive columnar storage;
- distinguish encoding, compression, physical types, and logical types;
- locate reader behavior in Hardwood and identify the right test boundary;
- investigate a reader bug without guessing at the responsible layer.

## How to use the course

Read the chapters in order. Each chapter assumes the vocabulary and invariants
from earlier chapters.

For each chapter:

1. Read the mental model and worked example without opening the answer key.
2. Perform the guided lab in the Hardwood checkout.
3. Answer the quiz in writing. Explanations matter more than short answers.
4. Compare your reasoning with the linked answer key.
5. Revisit any objective you could not explain without looking at the text.

The estimated 45–75 minutes per chapter is a study size, not a deadline. The
nested-data and pipeline chapters commonly need a second pass.

## Prerequisites

The course assumes:

- basic Java syntax, arrays, interfaces, records, and exceptions;
- familiarity with tables, columns, `NULL`, and SQL predicates;
- a shell from which you can inspect this checkout.

It does **not** assume familiarity with:

- hexadecimal notation or endianness;
- bit packing, variable-length integers, or compression;
- Parquet, Thrift, Arrow, Dremel, or storage-engine internals;
- Java virtual threads or concurrent pipelines.

Chapter 2 supplies the binary foundation used by later chapters.

## Lab setup

Run commands from the Hardwood repository root. In this checkout, that is the
parent of this course directory:

```shell
cd ..
```

Check the tools before a lab needs them:

```shell
java --version
docker --version
./mvnw --version
```

Hardwood requires Java 25 or newer to build. Its JARs run on Java 21 or newer.
The complete test suite uses Docker through Testcontainers. Do not install or
upgrade anything merely because a later optional lab mentions it; each lab
states its actual requirements.

Project rules require Maven commands to use the wrapper and to detect deadlocks
after 180 seconds. On GNU/Linux, a focused test therefore looks like:

```shell
timeout 180s ./mvnw -pl core -Dtest=MalformedMetadataFileTest test
```

If your platform lacks GNU `timeout`, use an equivalent command runner with a
180-second limit. The course never requires downloading an arbitrary Parquet
file: it uses the checked-in fixtures under
`core/src/test/resources/`.

The CLI labs assume a built CLI. You can build the project with:

```shell
timeout 180s ./mvnw -Dquick package
```

The public contribution guide asks you to run a clean project-wide
verification before pushing:

```shell
timeout 180s ./mvnw clean verify
```

## Course map

- [Chapter 1 — From SQL Tables to Columnar Storage](chapters/01-columnar-storage.md)
- [Chapter 2 — Binary Tools for a Java Reader](chapters/02-binary-toolkit.md)
- [Chapter 3 — The Physical Parquet Hierarchy](chapters/03-physical-layout.md)
- [Chapter 4 — The Footer and Thrift Compact Protocol](chapters/04-footer-and-thrift.md)
- [Chapter 5 — Schema, Physical Types, and Logical Types](chapters/05-schema-and-types.md)
- [Chapter 6 — Pages, Headers, Compression, and CRC](chapters/06-page-anatomy.md)
- [Chapter 7 — PLAIN Values and Nullability](chapters/07-plain-and-nulls.md)
- [Chapter 8 — RLE/Bit Packing and Dictionaries](chapters/08-rle-and-dictionaries.md)
- [Chapter 9 — Delta, Byte-Stream Split, and Codecs](chapters/09-encodings-and-codecs.md)
- [Chapter 10 — Nested Data and Dremel Levels](chapters/10-nested-data.md)
- [Chapter 11 — Opening a File in Hardwood](chapters/11-opening-a-file.md)
- [Chapter 12 — Planning I/O and Finding Pages](chapters/12-io-planning.md)
- [Chapter 13 — Decoding a Page](chapters/13-page-decoding.md)
- [Chapter 14 — Assembly, Concurrency, and Reader APIs](chapters/14-assembly-and-concurrency.md)
- [Chapter 15 — Skipping Work and Contributing Safely](chapters/15-pushdown-and-contributing.md)

## Progress checklist

- [ ] 1. I can explain why column projection is possible.
- [ ] 2. I can read a little-endian integer and compute a bit width.
- [ ] 3. I can distinguish a row group, column chunk, and page.
- [ ] 4. I can explain why a reader begins at the end of the file.
- [ ] 5. I can separate physical type, logical type, and repetition.
- [ ] 6. I can describe the V1 and V2 page-body boundaries.
- [ ] 7. I can explain why a page may have fewer encoded values than level entries.
- [ ] 8. I can hand-decode an RLE run and explain dictionary indices.
- [ ] 9. I can choose the relevant decoder family from metadata.
- [ ] 10. I can derive definition and repetition levels for a small list.
- [ ] 11. I can trace Hardwood's footer-read path.
- [ ] 12. I can contrast indexed and sequential fetch plans.
- [ ] 13. I can trace `PageDecoder.decodePage` in the correct order.
- [ ] 14. I can explain Hardwood's back-pressure chain.
- [ ] 15. I can place a failing test at the layer where an invariant broke.

## One picture to keep updating

Start with this deliberately simple flow. Every chapter adds detail to one
box, but the direction remains stable:

```text
SQL-like records
      |
      v
schema tree -----> leaf columns
                       |
                       v
file -> row groups -> column chunks -> pages
                                      |
                                      v
                           levels + encoded values
                                      |
                                      v
                             typed Java arrays
                                      |
                                      v
                            rows or column batches
```

When debugging, move through the picture in the direction of data and name the
first broken invariant. For example, a wrong logical timestamp can originate
after correctly decoded bytes; an unexpected end of input can originate before
logical conversion. Similar symptoms do not imply the same layer.

## Source-of-truth order

Parquet has a specification; Hardwood is one implementation. Keep these roles
separate:

1. The
   [Apache Parquet format repository](https://github.com/apache/parquet-format)
   and its `parquet.thrift` define the interoperable format.
2. The
   [Apache file-format documentation](https://parquet.apache.org/docs/file-format/)
   explains the format alongside that Thrift definition.
3. Hardwood's completed design
   [`_designs/PARSING_PIPELINE_V2.md`](../_designs/PARSING_PIPELINE_V2.md)
   records the intended v2 component model and its rationale. Some operational
   details have since changed, including chunk sizing, validity storage, and
   reader construction.
4. Current source and tests are the truth about what this checkout does.
5. [`ARCHITECTURE.md`](../ARCHITECTURE.md) is useful orientation, but some
   reader class names there are stale. In particular, verify names against
   `RowGroupIterator`, `PageSource`, `ColumnWorker`, and `PageDecoder`.

The course labels facts as **format rule** or **Hardwood choice** when the
distinction is easy to blur.

## Compact glossary

**Batch**  
A bounded group of decoded records or column values handed toward a consumer.
It is a runtime concept, not an on-disk Parquet unit.

**Column chunk**  
The contiguous data for one leaf column in one row group.

**Compression codec**  
A reversible byte-to-byte transformation such as Snappy, Gzip, or Zstandard,
applied to page data after value encoding.

**Definition level**  
For one encoded position, how far down an optional/repeated schema path is
defined. It carries null and empty-container structure.

**Encoding**  
A type-aware representation of values or levels, such as PLAIN,
RLE_DICTIONARY, or DELTA_BINARY_PACKED.

**Footer / file metadata**  
The Thrift-encoded structure near the end of a Parquet file that describes the
schema, row groups, column chunks, offsets, codecs, statistics, and optional
indexes.

**Leaf column**  
A primitive-valued path at the edge of the schema tree. Parquet stores data for
leaves, not group nodes.

**Logical type**  
An interpretation layered over a physical representation, such as UTF-8 text
over `BYTE_ARRAY` or a timestamp over `INT64`.

**Page**  
An independently headed and usually independently compressed region inside a
column chunk. Dictionary pages and data pages have different jobs.

**Physical type**  
The primitive on-disk value family: `BOOLEAN`, `INT32`, `INT64`, `INT96`,
`FLOAT`, `DOUBLE`, `BYTE_ARRAY`, or `FIXED_LEN_BYTE_ARRAY`.

**Projection**  
Reading only selected columns.

**Repetition level**  
For nested repeated data, the schema depth at which the current encoded
position continues the previous record rather than beginning a new one.

**Row group**  
A horizontal partition of records whose leaf columns are stored as adjacent
column chunks.

**Thrift Compact Protocol**  
The binary protocol used for Parquet metadata structures and page headers.

## After the course

Before changing code, read [`CONTRIBUTING.md`](../CONTRIBUTING.md), choose an
existing issue, and reproduce a bug with a failing test. Use
[`FORMAT_COVERAGE.md`](../FORMAT_COVERAGE.md) and
[`ROADMAP.md`](../ROADMAP.md) to understand support boundaries; do not infer
support from the existence of an enum constant alone.
