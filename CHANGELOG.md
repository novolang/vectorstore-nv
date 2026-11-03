# Changelog

All notable changes to vectorstore-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-11-03

### Changed

- File handles are bound with `let Some(fd) = ... else`, the keyword-index rebuild propagates with `!`, and the fused-hit source is chosen by a `match` over which halves ranked it; no public signature or result changes.

### Known follow-ups

- The `VsStore` mutators (`vs_upsert`, `vs_delete`) take `self` by value and write through a `var shared = self` alias in `VsMemory` and `VsFileStore`.  Declaring them `var self` would remove the aliases, but it is a breaking change: a caller holding a `let`-bound store could no longer call them (E2036), so it waits for the next breaking release and is not made in a patch.

This release requires novo 0.23.0 or later.

## 0.1.0 — 2026-10-11

The first implementation of the interface published as 0.0.1: every
module has a body.  Documents, collections and filters are values and
evaluating one is arithmetic; what performs is the file (`[fs]`) and the
two servers (`[net]`).  **No public declaration was added, removed,
renamed or narrowed** — every signature line and every effect row is
byte-identical to 0.0.3, so a caller on that version upgrades by changing
one version number.

### Behaviour the interface left open

- **The search plans.**  `VsAuto` resolves to `VsPreFilter` when the
  filter's estimated match count over a 256-document sample is at or
  below `pre_filter_max` (1000), and to `VsFilteredTraversal` above it.
  A post-filter is never chosen automatically, because it is the plan
  whose failure — answering fewer than k — is silent.  `ef_for` is the
  query's own `ef` when it set one and hnsw-nv's `default_ef` otherwise.
- **The id table.**  A document's slot in `docs` is its hnsw-nv label.
  A NEW id takes a new slot and never a marked one — `insert_replacing`
  would relabel a removed document's slot and hand its vector back under
  a stranger's name.  An upsert of a live id marks its own slot and
  re-inserts into it, and a delete marks the slot and records the id in
  `deleted`, so the tombstone is enforced on the graph walk and on every
  path that reads `docs` alike.  `compact` rebuilds the index over the
  live documents, which is the only call that reclaims the space.
- **`VsNone` on qdrant is a `must`, not a `must_not`.**  A qdrant
  `must_not` is `(NOT A) AND (NOT B)`, and a condition on a field a point
  does not have is not satisfied — so the impossible condition written as
  a negation came back as every point qdrant had, which is `VsAll`, the
  one answer `VsNone` exists to not give.  Written as a `must` it
  excludes them.
- **A score is a similarity everywhere**, including out of a server: the
  exact paths rank by `embsim.as_distance` under `VsConfig.metric`, and
  hnsw-nv's minimum distance and a server's own distance are converted at
  the one boundary each.
- **The file carries the index** rather than rebuilding it, and the
  header carries the model and the metric.  `from_bytes` checks the
  index's own dimension and metric against the header before it trusts
  either, and `save` writes a temporary name in the same directory and
  renames over the target.
- **The remote transport** is HTTP/1.1 with `Connection: close`, one
  `send` and a read to the socket's end, over `std.tls` for an `https://`
  base URL.  `VsSearchOutcome.exhaustive` is false for every filtered
  remote query, because neither server reports whether its filter cut the
  search short.

### Changed

- `x == false` written as `not x` in seven test sites and one README
  snippet, which is what `novo.lint`'s `redundant-bool` asks for and
  cannot change a predicate.  All eight are pre-existing — confirmed
  against the interface release with `git show HEAD:<file>` (three in
  `vsvalue_tests.nv`, three in `vsio_tests.nv`, one in
  `vssearch_tests.nv`, one in `README.md`) — not introduced here.
- A local `path` in `vectorstore_edge_tests.nv` renamed `target`: the
  compiler prints `W0002 binding 'path' shadows stdlib module 'path'`
  where `novo.lint` answers `count 0` — the two do not agree, and the
  compiler is the one a reader meets.

### Fixed

- **A section length near the integer ceiling is a bad file.** `vsfile.from_bytes` and `vsfile.read_header` summed an offset and a stated length before comparing it with the buffer, so a crafted length trapped on overflow, and a header length read from disk was allocated before it was checked. Both now compare by subtraction and answer `VsBadFile`.

### Known

- **chroma has no operator for three of the clauses.**  `VsPrefix`,
  `VsExists` and `VsMissing` are accepted by the compiler and refused at
  run time by `vshttp.encode_filter(VsChroma, …)` as
  `VsUnsupportedQuery` naming the clause; qdrant carries all three.  A
  filter that is machine-generated for one server and run against the
  other can therefore be refused after it is built, which is the one
  failure rule 10's closed language was meant to prevent — disclosed
  rather than translated into an approximation that means something else.
- **The metric lives in two places and nothing compares them.**
  `VsConfig.metric` is what the exact paths rank by and
  `VsConfig.params.metric` is what the graph walks by.  A `VsConfig`
  built by hand with the two disagreeing is not refused; a collection
  loaded from a file is (the header and the index are checked).  Build
  both from one metric with `hnswparam.default_params(dim, metric)`.
- **A remote result is not exhaustive, and says so.**  `exhaustive` is
  false for every filtered remote query — a loss of information rather
  than a defect, and a caller who needs the guarantee runs the query
  against a local collection.
- **chroma's distances are read positionally, and a truncated list
  scores 0.0.**  `chroma_hits` pairs row `i` with distance `i`; a reply
  whose `distances` list is shorter than its `ids` list gives the
  unpaired rows a distance of 0.0 — the nearest possible — rather than
  refusing the reply.  The two lists are parallel in chroma's own
  answer, so this needs a server that broke its contract; the corner is
  disclosed rather than refused late.
- **A chunked HTTP reply is refused, not dechunked.**  Both servers send
  `Content-Length` for the JSON bodies this package asks for; a reply in
  chunked encoding arrives as its chunk headers and is refused as
  malformed JSON rather than silently mis-read.
- **A workaround is in `vssearch`.**  `post_filtered` and `traversed`
  re-wrap `hnswsearch.search`'s `Result` and DESTRUCTURE the payload with
  a later `match` instead of binding it with `let rows = raw!`, which
  answers an empty list for a list payload in a module other than the
  compilation's entry file — every filtered search answered zero hits
  with every verdict green.  Filed as
  `memory-perceus/a-let-bound-bang-of-a-re-wrapped-result-answers-an-empty-list`;
  each site carries a comment naming it.
- **A workaround is in `vsdoc`.**  `upsert` sends its index through a
  `place` helper and binds the answer once, rather than rewriting one
  local with two `x = f(x)!` assignments, because a failing `!`
  assignment does not release the value the previous one left there —
  the collection lost an index per refusal with every verdict green.
  Filed as
  `memory-perceus/a-failing-bang-assignment-leaks-the-value-a-previous-bang-assignment-left-in-the-local`;
  the site carries a comment naming it.
- **A workaround is in `vsfilter`.**  `value_of` reads a number's own
  text (`json.stringify`, then `str.to_int` and `str.to_float` on it)
  instead of `json.to_int`, which truncates a double on the compiled
  leg (4.5 becomes 4) where the interpreter answers `None` — a filter
  would have compared a different number than the document holds.
  Filed as
  `stdlib/json-get-int-and-to-int-truncate-a-double-where-the-interpreter-answers-none`;
  the site and the test that pins it name the filing.
- **A workaround is in `vshttp`.**  The two places a missing JSON
  member is defaulted — a qdrant point's payload, a chroma row's
  metadata — use a `match` and the three-argument `list.get`, because
  the miss path of `json.get(v, k) ?? json.object([])` leaks the object
  it evaluates: five cells per run of `tests/vshttp_tests.nv`, every
  verdict green, exit 23.  Filed as
  `memory-perceus/a-null-coalesce-fallback-over-list-get-first-or-json-get-is-leaked-on-the-miss-path-and-dropped-in-one-tail-shape`;
  both sites carry a comment naming it.

### Tests

- 157 tests in six suites, green and leak-free.  The four oracle
  suites are the interface's own — 48 in `vsvalue`, 30 in
  `vssearch`, 19 in `vsio`, 14 in `vsstore`.  The runs are compiled
  with the RC leak checker on and every exit 23 was chased to zero,
  which is where three of the workarounds above came from.
- The pins that carry the design were mutation-tested on a scratch
  copy, and one survived: putting qdrant's `VsNone` back in `must_not`
  and hard-coding the exact path's metric to cosine each go red, but
  the id-table pin did not catch `insert_replacing` where a new id must
  take a NEW slot — the case that distinguishes them is
  delete → unrelated add → re-add, which nothing exercised.
  `test_a_stranger_in_a_freed_slot_does_not_take_the_removed_id_with_it`
  is that case, and it is red under that mutation.
- `tests/vectorstore_edge_tests.nv` is the fifth suite (12 tests).  It
  pins the five ways a store like this corrupts an answer while every
  verdict stays green: the id mapping across a delete, an unrelated
  add and a re-add, a
  case where cosine, dot and Euclidean rank one corpus three different
  ways, every query shape asked for a deleted document, **recall
  measured against a brute-force scan** rather than asserted: 1.0 over 20
  deterministic queries, k=10, over 1200 vectors of 16 dimensions, with
  the number printed by the run, and **the half-written file** — a
  collection whose replacement write fails leaves the one that was
  already there, and the temporary name behind.  The suite also checks
  that each server translation MEANS what `vsfilter.matches` means, by
  evaluating both wire encodings with a reader written from each
  server's own documented semantics, and that a keyword-only collection
  (`VsConfig.dim` zero) takes documents with no vector while a
  dimensioned one refuses the same document.
- `tests/vshttp_tests.nv` is the sixth (34 tests): both dialects'
  request bodies asserted as exact strings, every filter translation
  checked against the clause it came from (including the three clauses
  chroma has no operator for, refused by name), the reply decoders (a
  malformed reply, a page inside an `ids` list, a reply cut to its
  stated `Content-Length`), the point-id mapping (`is_uuid_shape`, the
  version and variant nibbles), the scroll cursor and the dialect's own
  error body — plus two `[io, net]` tests that drive a refused
  connection through every method of the remote store, and one that
  **is** the server: a listener on a port the kernel picks, driven from
  a worker cell, serving every request of both dialects and asserting
  each request line the bytes actually carry (the prefixed base URL
  included), so the transport is pinned over a socket rather than over
  a string.
- **Coverage measured at publish** is 99% of 1832 instrumented lines
  under `src/` over the six suites, with 14 regions excused by
  `// cov: skip` and no stale marker — each excused region is an arm no
  test can reach (a callee that cannot refuse, a reader whose key came
  from the same object, the TLS half of the transport, an empty
  `net.recv` chunk, a write that cannot fail on a fresh connection), and
  every other `src/` line the suites ever missed was closed with a test
  instead: the transport lines by the loopback server above, and the
  malformed-reply and nothing-to-delete arms by served bodies of
  exactly those shapes.

## 0.0.4 — 2026-10-25

A file that names a sibling module's type, function or constant now has the `use` line for that module (E2033, E2003).  No signature and no behaviour changed.

## 0.0.3 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- hnsw-nv: `^0.0.1` to `^0.1.0`.
- embeddings-nv: `^0.0.1` to `^0.1.0`.
- unicode-nv: `^0.0.1` to `^0.1.3`.

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
