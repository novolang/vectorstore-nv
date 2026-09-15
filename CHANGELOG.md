# Changelog

All notable changes to vectorstore-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `vssearch` — `VsFilterPlan` with the three honest filter strategies,
  `VsAuto`'s rule published as a threshold, and `VsSearchOutcome` with
  `exhaustive`, the plan that ran, and what it examined.  `exact_search`
  as the oracle and `recall` as the measurement.
- `vsdoc` — the document and the collection as values, with the model
  recorded and checked, an upsert that says what it replaced, and a
  delete that marks with `compact` beside it.
- `vsfilter` — a small typed predicate language, with three
  constructions deliberately absent and each argued.
- `vsbm25` — Okapi BM25 over unicode-nv's tokens, with Robertson and
  Walker's constants named as theirs and BM25+ off by default.
- `vsrank` — reciprocal rank fusion over `VsRanking`, which carries ids
  and no scores.
- `vsstore` — `VsStore[e]`, the trait a memory collection, a file and a
  server all sit behind, with `VsMemory` at `[]`.
- `vsfile` — the format at `[]` and the filesystem at `[fs]`, with an
  atomic rename and the model in the header.
- `vshttp` — the qdrant and chroma dialects, codecs at `[]` and client
  at `[net]`.
- `vsfault` — thirteen variants.

### Known

- **`VsFilterPlan` is the load-bearing interface.**  A post-filter
  silently answers fewer than k, a pre-filter is exact and linear, and
  a filtered traversal can disconnect the matches from the entry point
  and answer nothing.  `VsAuto` never chooses the post-filter, because
  it is the one whose failure is silent.
- **`exhaustive` is published on every result.**  "Three documents
  match" and "three of the ten I looked at match" are different facts,
  and no other vector store answers which one it gave.
- **The score is a similarity in every path.**  hnsw-nv compares by
  minimum distance and a ranking sorts by maximum similarity; the two
  remote dialects disagree with each other as well.  Getting it
  backwards returns the furthest documents and looks like it works.
- **Fusion merges ranks, not scores.**  A BM25 score and a cosine
  similarity are not on a scale, so `VsRanking` carries ids and has
  nowhere to put one.  The cost — the lost margin — is stated.
- **The collection records its embedding model**, and a mismatch is
  refused at upsert, at search and at load.  Two models of the same
  dimension are two spaces, and nothing else catches it.
- **The filter language omits three things on purpose**: arbitrary
  expressions, negation over a subtree, and implicit type coercion.
  Each would be a run-time surprise where the compiler could have said
  so.
- **The file is not a database.**  Whole-file writes, an atomic rename,
  and no concurrent writer; a corpus that changes faster wants
  `vshttp`.
- **Four differences between the two servers are disclosed rather than
  hidden**: the point id, the unavailable `exhaustive`, chroma's
  missing sparse vectors, and the fact that a remote store is not a
  value.
- **This package embeds nothing.**  A vector is a `[Float]` the caller
  supplies.
- **novoagent's retrieval would take four things** — the receipt as
  metadata, `exhaustive`, hybrid search, and the in-memory store — and
  would not take the eviction policy.  The README has the argument.
- **Three `core` dependencies**: hnsw-nv, embeddings-nv, unicode-nv.
- **No device claim.**  The package is `host`.
