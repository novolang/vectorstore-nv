# vectorstore-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

A document store with vectors.  Documents with text, metadata and an
embedding; a typed metadata predicate language; k-nearest search
through hnsw-nv with the filter strategy as a **named decision rather
than a hidden one**; BM25 over unicode-nv's tokens; hybrid retrieval
merged by reciprocal rank fusion; a file on disk; and the qdrant and
chroma HTTP dialects behind the same trait.

It embeds nothing.  A vector is a `[Float]` the caller supplies —
llm-client-nv, ollama-nv or an in-process model produces it — because
a store that embedded would have taken `[net]` on every insert and
picked the caller's model for them.

## Adding it, and checking it

```bash
novo pkg add vectorstore-nv    # into your novo.toml
novo pkg build                 # type- and effect-check the package
novo test --isolate tests/vssearch_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: vectorstore-nv.<module>.<fn>`.
They turn green one at a time as bodies land.

## The one example that will work

```novo
use vsdoc
use vsfilter
use vssearch
use vsstore

// Retrieve, and know whether the answer is the whole truth.
fn recent(s: VsMemory, query: [Float]) -> Result<VsDocResults, VsFault>
    let q = vssearch.with_plan(
                vssearch.with_filter(vssearch.query(query, 10),
                                     vsfilter.is_clause(VsGt("year", VsInt(2020)))),
                VsPreFilter)

    let r = vsstore.retrieve(s, q)!

    // READ THIS.  A post-filtered search answers fewer than k without
    // saying so, and the result looks exactly like a correct one.
    if r.outcome.exhaustive == false
        println(vssearch.explain(r.outcome))
    Ok(r)
```

No effect row on that function, and that is the trait working: the
in-memory store performs nothing.  The same code against a file is
`[fs]` and against a server is `[net]`.

## The layer, and why

`host`, and six of the nine modules declare nothing.

| module | row | why |
| --- | --- | --- |
| `vsfault` | `[]` throughout | a fault is a value |
| `vsdoc` | `[]` throughout | a collection is a value; every call takes one and returns one |
| `vsfilter` | `[]` throughout | a filter is a value and evaluating one is arithmetic |
| `vsbm25` | `[]` throughout | an index is a value and scoring is arithmetic |
| `vsrank` | `[]` throughout | fusing two rankings is arithmetic |
| `vssearch` | `[]` throughout | a search over a collection value |
| `vsfile`'s format half | `[]` | `to_bytes` / `from_bytes` / `peek_header` |
| `vsfile`'s file half | `[fs]` | `save`, `load`, `open`, `flush` |
| `vshttp`'s codec half | `[]` | `encode_search`, `decode_search`, `encode_filter` |
| `vshttp`'s client half | `[net]` | `std.tls`'s own row |
| `vsstore.retrieve` and friends | `[e]` | effect-POLYMORPHIC: whatever the store costs |

Every codec is published beside its effectful call, so a caller that
keeps collections in a database column, or drives its own HTTP client,
uses this package without either effect row.

## The load-bearing interface

**`VsFilterPlan`, and `VsSearchOutcome.exhaustive` beside it.**

A filter and an approximate index do not compose the way everybody
assumes.  HNSW is a graph whose search walks from an entry point toward
the query, and its guarantee is about a walk over *all* the nodes.  A
metadata filter changes which nodes count, and there are only three
honest ways to apply it:

- **Post-filter.**  Search for k, then drop what does not match.
  **Silently answers fewer than k.**  A caller asking for 10 documents
  with `year > 2020` gets 3 — not because only 3 match the filter, but
  because only 3 of the top 10 by vector did.  There may be a thousand
  matching documents and this found three.  Every naive implementation
  does this, and the result is a value that looks exactly like a
  correct one.
- **Pre-filter.**  Evaluate the filter over the corpus, then compare
  the query against the survivors exactly.  Always right, and linear in
  the number of survivors — cheap for a narrow filter, a full scan for
  a broad one.
- **Filtered traversal.**  Walk the graph, skipping rejected nodes.
  hnsw-nv's `search_filtered` is this.  Fast, and its failure is the
  interesting one: the graph is connected *through* the nodes that were
  skipped, so a selective filter can disconnect the matching documents
  from the entry point and the search answers **nothing at all**,
  though thousands match.

So the plan is a value the caller passes; `VsAuto` chooses by estimated
selectivity with the rule written down (`vssearch.plan_for`, threshold
published as `pre_filter_max`) rather than tuned; **a post-filter is
never chosen automatically**, because it is the plan whose failure is
silent; and **every result says which plan ran and whether it was
exhaustive**.

That last field is the one no other vector store answers.  A result of
three documents is either "three documents match" or "three of the ten
I looked at match, and I do not know how many others do".  A caller
that cannot tell those apart cannot tell a working retrieval pipeline
from a broken one.

## The score direction, which is the other silent one

hnsw-nv answers **distances** and compares by minimum; a ranking sorts
by **maximum similarity**.  `embsim.as_distance` is the one line
between the two conventions, and getting it backwards produces a store
that returns the *furthest* documents and looks like it works — the
answers are documents, they are ordered, and they are wrong.

So `VsHit.score` is a similarity in every path through this package,
higher being closer, with `distance` beside it for a caller measuring
recall against the index's own numbers.  `vshttp` normalises the same
way: qdrant answers a similarity for cosine and a distance for
euclidean, chroma answers a distance always, and both come out here as
similarities.

## Reciprocal rank fusion merges ranks, not scores

A BM25 score is an unbounded sum of per-term weights over a particular
corpus — 14.2 is a big number for one index and a small one for
another, and adding a document changes every score in it.  A cosine
similarity is in [-1, 1] and means the same thing everywhere.  **The
two numbers are not on a scale; there is no exchange rate between
them.**

So every `alpha * vector_score + (1 - alpha) * keyword_score` a
retrieval pipeline contains is arithmetic on incomparable units, and it
works exactly as well as the normalisation somebody guessed at — which
is why tuning `alpha` never feels like it converges.  Min-max
normalising each result set is worse in a specific way: it makes the
top score 1 in *every* result set, so a query where the keyword half
found nothing relevant still contributes a full-strength 1.0 at the
top.

Ranks are comparable.  First is first in both lists.  RRF scores a
document as the sum of `1 / (k + rank)` across the lists that found it
— no normalisation, no weights to tune, no corpus dependence — and a
document found second and third beats one found first and nowhere.
Cormack, Clarke and Buettcher's `k` of 60 is the default and is named
as theirs.  `VsRanking` carries **ids and no scores**, so a caller
cannot hand the fuser a score it wanted fused.

The cost is stated too: fusing by rank throws away the **margin**.  A
vector search whose top hit is far ahead of its second and one whose
top ten are indistinguishable produce the same ranks.  `weights` is the
escape for a caller who has measured their own corpus, and
`VsSource.VsFromBoth` is what an explanation shows instead.

## The collection records the model it was built with

Two embedding models of the same dimension are two different spaces.
Querying a collection built with one using a vector from the other
produces distances that are arithmetic, plausible and meaningless, and
every document comes back in an order that has nothing to do with the
query.  **Nothing else in a retrieval pipeline catches it** — the
results look like results.

So `VsConfig.model` is a string the caller chooses, compared for
equality and nothing more, and `VsModelMismatch` is refused at
`upsert`, at `search`, and at `load` — the file header carries it, so a
collection loaded into the wrong program is refused before a vector is
read.

## The filter is a small typed language

Every remote store in this space takes a filter as a nested JSON
object, and the temptation is to let the caller write one — which makes
every filter a run-time parse, a typo a 400 from a server, and the two
dialects' incompatible spellings the caller's problem.

A value instead.  What is deliberately *not* in it:

- **No arbitrary expressions.**  Everything here translates to *both*
  dialects; a construct only one supports would make `vshttp` refuse at
  run time for a filter the compiler accepted.
- **No `not` over a subtree**, only over one clause.  `not (year >
  2020)` is true for a document with no `year` at all, which is almost
  never what the person meant; the one-clause form keeps that visible
  and `VsMissing` is the explicit way to ask.
- **No implicit coercion.**  `"2021" > 2020` is `VsBadFilter`.  Both
  remote dialects silently answer false, which is indistinguishable
  from "no documents matched".

`vsfilter.matches` is this package's definition of what a filter means,
and each remote translation is tested against it — the same filter and
the same document through all three.

## What novoagent's retrieval would take

orbit/novoagent has no retrieval today.  Its context manager is an
eviction policy over a monotonically growing transcript: the system
prompt and the task are pinned, the last K turns are kept verbatim,
older tool observations collapse to a one-line receipt, and the run
ends as `budget_exhausted` when the ceiling is crossed.  The
observations it collapses are mostly documentation pages — learning the
language *is* the task there — and the collapse is lossy on purpose:
the fact of the call survives and the payload does not.

What this package changes is where the payload goes. A collapsed
observation's text becomes a document; the receipt stays in the
transcript as it does now; and the agent gets a retrieval tool that
searches what it has already read. Four things it would take:

- **`VsDoc.metadata` as the receipt**: the tool name, the arguments
  and the turn number, so `VsFilter` can scope a search to this run, to
  documentation only, or to observations since the last compiler error.
- **`exhaustive`.**  An agent that retrieved three fragments and
  concluded the answer is not in its memory has made a decision on
  incomplete information without knowing it.  The one field that makes
  "I found three" and "I found three of the ten I looked at" different
  is the one an agent loop most needs.
- **Hybrid search, not vector search.**  The agent's queries are half
  natural language and half identifiers — a function name, an error
  code, a module. `ORA-01555` and `E2004` embed to nearly the same
  place as their neighbours and mean something specific; BM25 tells
  them apart because it never looked at meaning.
- **The in-memory store.**  An agent run is minutes long and its corpus
  is what it has read, so the collection is a value in the loop's own
  state, `[]` throughout — no file, no server, and testable without
  either.

What it does **not** take is the eviction policy: which turns are
pinned and what collapses is the agent's decision about its own
attention, and a retrieval index is a place to put what was collapsed
rather than a reason to collapse less.

## The file is not a database

`save` writes the whole collection and `load` reads it.  No concurrent
writer, no partial update, no transaction.  `save` writes to a
temporary name in the same directory and renames over the target, so a
process that died halfway leaves the previous collection intact rather
than a truncated file — a rename within a directory is the only atomic
operation a filesystem offers, and a store whose whole state is one
file should use it.

A corpus that changes faster than it can be rewritten wants a server,
which is `vshttp`.  That line is stated here rather than discovered at
the size where it starts mattering.

The file carries hnsw-nv's serialised index rather than rebuilding it:
a hundred thousand vectors take minutes to index and milliseconds to
read.  `hnswio` has a magic, a version and stated widths, which is what
makes that safe — hnswlib's own raw memory dump has none of the three.

## Two servers, one trait, and what is not hidden

- **A document id is not the same thing.**  qdrant's point id is an
  unsigned integer or a UUID; chroma's is any string.  `vshttp.point_id`
  hashes a non-UUID id into a stable UUID and keeps the original in the
  payload — so a qdrant collection written by this package has an `id`
  payload field another client has to read, and that is said here
  rather than discovered.
- **Neither server reports whether a filtered search was exhaustive**,
  so `exhaustive` is false for every filtered remote query.  A loss of
  information rather than a defect, and a caller that needs the
  guarantee runs the query locally.
- **chroma has no sparse vectors.**  A hybrid query against it is
  `VsUnsupportedQuery` naming the server, not a vector search that
  quietly dropped its keyword half.
- **A remote store is not a value.**  `snapshot` is on the local store
  only; the remote equivalent is `vshttp.scroll`, and it is named
  differently because downloading a corpus one page at a time is a
  different thing.

## Dependencies

Three, all `core`:

- **hnsw-nv** — the index, and `hnswio`'s serialised form.
- **embeddings-nv** — `embsim.as_distance`, the one line between the
  index's minimum-distance rule and a ranking's maximum-similarity one.
- **unicode-nv** — word boundaries and case folding for BM25's tokens.

## Licence

Apache-2.0.
