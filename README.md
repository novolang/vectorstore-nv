# vectorstore-nv

A vector store keeps documents beside the vectors that stand for them, so
a program can ask for the documents nearest a query. This package is one in
novo-lang: documents with text and metadata, a typed metadata filter,
k-nearest search over
[hnsw-nv](https://novo-lang.org/packages/hnsw-nv), keyword search by
[BM25](https://en.wikipedia.org/wiki/Okapi_BM25), the two combined by
reciprocal rank fusion, a file on disk, and the
[qdrant](https://qdrant.tech/documentation/) and
[chroma](https://docs.trychroma.com/) HTTP APIs behind the same trait. It
is built on hnsw-nv,
[embeddings-nv](https://novo-lang.org/packages/embeddings-nv) and
[unicode-nv](https://novo-lang.org/packages/unicode-nv).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What it is

A **document** is an identifier, some text, some metadata as a JSON value,
and a **vector**: a list of numbers a model produced from the text. A
**collection** is a set of documents, the index over their vectors, and the
configuration both were built under.

Searching by vector is **approximate**. hnsw-nv walks a graph from an entry
point toward the query, so it finds the nearest documents almost always
rather than always. **Recall** is the fraction of the true nearest it
found.

A **filter** is a condition on a document's metadata, such as
`year > 2020`. A filter and an approximate index do not combine the way
they appear to, and there are three honest ways to apply one.

| Plan | What it does | What it costs |
| --- | --- | --- |
| `VsPostFilter` | search for k, then drop what does not match | answers fewer than k without saying so; there may be a thousand matching documents and this found three |
| `VsPreFilter` | evaluate the filter over the corpus, then compare the survivors exactly | always right; linear in the number of survivors |
| `VsFilteredTraversal` | walk the graph, skipping rejected nodes | fast; a selective filter can cut the matching documents off from the entry point, and the search answers nothing at all |

**BM25** is a keyword score: how well a document matches a query's words,
given how rare each word is in the corpus and how long the document is. It
never looks at meaning, which is why it finds an error code or a function
name that a vector search puts next to its neighbours.

A **hybrid** search runs both and merges them. **Reciprocal rank fusion**
merges by position rather than by score: a document's fused score is the
sum of `1 / (k + rank)` over the lists that found it.

Every comparison this package answers is a **similarity**, where higher is
closer.

| Quantity | Value |
| --- | --- |
| BM25 term-frequency saturation, `k1` | 1.2 |
| BM25 length normalisation, `b` | 0.75 |
| BM25+ lower bound, `delta` | 0.0 |
| Reciprocal rank fusion constant, `k` | 60 |
| File magic | `NVVS` |
| File format version | 1 |

Six of the nine modules declare no effects. `vsfile`'s file half is `[fs]`,
`vshttp`'s client half is `[net]`, and the store calls in `vsstore` cost
whatever the store they are handed costs. Every codec is published beside
its effectful call, so a caller that keeps collections in a database column
or drives its own HTTP client spends neither.

This package embeds nothing. A vector is a list of floats the caller
supplies.

## Install

```
novo pkg add vectorstore-nv
```

## Example

```novo
use vsdoc
use vsfilter
use vssearch
use vsstore
use hnswparam
use std.list

// Find the ten nearest documents published after 2020. No effect row:
// the in-memory store performs nothing. The same code over a file
// costs `[fs]` and over a server `[net]`.
fn recent(s: VsMemory, query: [Float]) -> Result<VsDocResults, VsFault>
    // Filter first, then compare exactly against what survives. That
    // plan is always right, and it is linear in the survivors.
    let q = vssearch.with_plan(
                vssearch.with_filter(vssearch.query(query, 10),
                                     vsfilter.is_clause(VsGt("year", VsInt(2020)))),
                VsPreFilter)
    let r = vsstore.retrieve(s, q)!
    Ok(r)

fn main() [io]
    match hnswparam.default_params(2, HnswCosine)
        Err(f) => println(f.message())
        Ok(p)  =>
            let c = vsdoc.collection("notes", vsdoc.config(2, HnswCosine, "demo-model", p))
            match recent(vsstore.memory(c), [1.0, 0.0])
                Err(f) => println(f.message())
                Ok(r)  =>
                    // A search that was not exhaustive may have missed
                    // matches, and its result looks like a complete one.
                    if r.outcome.exhaustive == false
                        println(vssearch.explain(r.outcome))
                    println("${list.len(r.docs)} document(s)")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: vectorstore-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `vsdoc` | A document, a collection's configuration, the collection itself, and the calls that add, remove and read documents. |
| `vsfilter` | The metadata predicate language: values, clauses, the combinators, and what a filter means over one document. |
| `vsbm25` | The keyword index: the tokenizer, the parameters, building and scoring, and the per-term numbers behind a score. |
| `vsrank` | Reciprocal rank fusion: the constant, the weights, the ranked lists, and the overlap between two of them. |
| `vssearch` | A query, the four filter plans, the result and what it says about itself, exact search, and the recall measurement. |
| `vsstore` | The store trait, the in-memory store, and the three calls that work over any store. |
| `vsfile` | The serialised collection, its header, and the four file operations. |
| `vshttp` | The two server dialects: their requests, their replies, their errors, and the client behind the same trait. |
| `vsfault` | The thirteen ways a store call fails, and whether each is worth retrying. |

## How to choose an entry point

**`vsstore.retrieve` works over any store.** It takes the trait, so the
same code runs against memory, a file and a server, and its effects are
that store's. `ingest` is the batched insert and `check_query` is the
check on its own.

**`vssearch.search` takes a collection value directly.** Use it when the
collection is in the program's own state and no store wrapper is wanted. It
declares no effects.

**One query type covers all three searches.** `vssearch.query` is a vector
search, `text_query` is keyword only, and `hybrid_query` is both. A caller
switching between them changes a value rather than a call.

**`vssearch.exact_search` compares against every document.** It is the
answer `recall` measures an approximate search against, and the right
choice for a collection of a few thousand documents.

**`vsfile.to_bytes` and `from_bytes` have no effects**, and `save` and
`load` are the same thing against a path. Take the first pair for a
collection kept in a database column, an object store or a test.

**`vshttp.encode_search` and `decode_search` have no effects**, and the
trait implementation on `VsHttp` is the same exchange over a socket.

## The rules a user needs

1. **Read `VsSearchOutcome.exhaustive` before acting on a result.** A
   result of three documents is either "three documents match" or "three of
   the ten I looked at match, and I do not know how many others do". It is
   true for a pre-filter and for an unfiltered search over a small
   collection, and false for a post-filter that hit its over-fetch limit
   and for a filtered traversal.
2. **A post-filter is never chosen automatically.** Its failure is the
   silent one: a caller asking for ten documents with `year > 2020` gets
   three, because only three of the top ten by vector matched, not because
   only three documents match. `VsAuto` chooses between a pre-filter and a
   filtered traversal, and a caller that wants a post-filter names it.
3. **`VsAuto`'s rule is published, not tuned.** When the filter's estimated
   selectivity puts the survivors below `vssearch.pre_filter_max`, it
   pre-filters, because an exact scan over a small set wins outright. Above
   that it traverses, because a linear scan over most of a corpus is what
   an index exists to avoid. `vssearch.plan_for` answers what it would
   choose and `estimate_matches` is the estimate.
4. **`VsSearchOutcome.plan` is the plan that ran, and `VsAuto` never
   appears in it.** `VsAuto` is a request and this is the answer.
5. **A score in this package is a similarity and higher is closer.**
   hnsw-nv answers distances and compares by minimum; `embsim.as_distance`
   is the line between the two conventions. Getting it backwards produces a
   store that returns the furthest documents and looks like it works.
   `VsHit.distance` is the index's own number, for measuring recall.
6. **The collection records the model its vectors came from.** Two models
   of the same dimension are two different spaces, and querying one with a
   vector from the other gives distances that are arithmetic, plausible and
   meaningless. `VsConfig.model` is free text compared for equality, and
   `VsModelMismatch` is refused at upsert, at search and at load. The file
   header carries it, so a collection loaded into the wrong program is
   refused before a vector is read.
7. **A coordinate that is not a finite number is refused on the way in.**
   Downstream it is silent: one such coordinate makes every distance
   involving that document not a number, and the document is simply never
   the nearest.
8. **Reciprocal rank fusion merges ranks, and that is deliberate.** A BM25
   score is an unbounded sum over a particular corpus, and adding a
   document changes every score in it. A cosine similarity is between −1
   and 1 and means the same thing everywhere. There is no exchange rate
   between the two, so `alpha * vector + (1 - alpha) * keyword` is
   arithmetic on incomparable units. `VsRanking` therefore carries ids and
   no scores.
9. **Fusing by rank throws away the margin.** A vector search whose top hit
   is far ahead of its second and one whose top ten are indistinguishable
   produce the same ranks. `VsFusion.weights` is the escape for a caller
   who has measured their own corpus, and `VsSource.VsFromBoth` is what an
   explanation shows instead.
10. **A filter is a value, not a JSON document.** Everything in the
    language translates to both server dialects, so a filter the compiler
    accepted cannot be refused at run time by one of them. A substring
    match and a regular expression are absent for that reason.
11. **Negation applies to one clause, not to a subtree.** `not (year >
    2020)` is true for a document with no `year` at all, which is almost
    never what was meant. `VsMissing` is the explicit way to ask for an
    absent field, and it is separate from a negated `VsExists` because
    absent and JSON null are two states.
12. **There is no implicit coercion in a filter.** Comparing the string
    `"2021"` with the number 2020 is `VsBadFilter`. Both server dialects
    silently answer false, which is indistinguishable from no document
    matching.
13. **`VsAll` and `VsNone` are different.** "The user selected no facets"
    and "the user selected an impossible combination" are different
    questions, and a store that answered `VsAll` for the second returns the
    whole corpus.
14. **`vsfilter.matches` is what a filter means.** Each server translation
    is tested against it, with the same filter and the same document
    through all three.
15. **A hybrid search on a collection with no keyword index is refused.**
    `VsUnsupportedQuery` names what was missing, which is better than a
    hybrid search that quietly becomes a vector search.
    `vsdoc.with_keyword_index` turns one on.
16. **The file is not a database.** `save` writes the whole collection and
    `load` reads it: no concurrent writer, no partial update, no
    transaction. `save` writes to a temporary name in the same directory
    and renames over the target, so a process that died halfway leaves the
    previous collection intact. A corpus that changes faster than it can be
    rewritten wants a server.
17. **The file carries the index rather than rebuilding it.** A hundred
    thousand vectors take minutes to index and milliseconds to read. That
    is safe because hnsw-nv's serialised form has a magic, a version and
    stated field widths.
18. **A document id is not the same thing in both servers.** qdrant's point
    id is an unsigned integer or a UUID and chroma's is any string.
    `vshttp.point_id` hashes a non-UUID id into a stable UUID and keeps the
    original in the payload, so a qdrant collection written by this package
    has an `id` payload field another client has to read.
19. **Neither server reports whether a filtered search was exhaustive**, so
    `exhaustive` is false for every filtered remote query. A caller that
    needs the guarantee runs the query locally.
20. **chroma has no sparse vectors.** A hybrid query against it is
    `VsUnsupportedQuery` naming the server.
21. **A remote store is not a value.** `vsstore.snapshot` is on the local
    store only. `vshttp.scroll` is the remote equivalent, and it is named
    differently because downloading a corpus a page at a time is a
    different thing.
22. **A delete marks rather than removes.** `vsdoc.deleted_count` is how
    many are marked and `vsdoc.compact` rebuilds without them, taking the
    uniforms hnsw-nv's insertion needs.

## What is not included

- **Embedding.** A vector is a list of floats the caller supplies. A store
  that embedded would take `[net]` on every insert and choose the caller's
  model for them.
- **Concurrency.** A collection is a value. Two writers are two values.
- **A transaction log.** See rule 16.
- **A substring match or a regular expression in a filter.** Neither
  translates to both server dialects.
- **Arbitrary filter expressions.** See rule 10.
- **A microcontroller build.** The package is `host`: it has a file and a
  socket in it.

## Related packages

- [hnsw-nv](https://novo-lang.org/packages/hnsw-nv) is the approximate
  index and its serialised form. Its `search_filtered` is what
  `VsFilteredTraversal` calls, and its distances are what `VsHit.distance`
  carries.
- [embeddings-nv](https://novo-lang.org/packages/embeddings-nv) is the
  arithmetic that produces a vector from a model's output: pooling,
  normalisation, truncation and quantisation. `embsim.as_distance` is the
  one line between its convention and the index's.
- [unicode-nv](https://novo-lang.org/packages/unicode-nv) gives BM25 its
  word boundaries and case folding. A keyword index that split on ASCII
  spaces would index a Japanese document as one term.
- [llm-client-nv](https://novo-lang.org/packages/llm-client-nv) and
  [ollama-nv](https://novo-lang.org/packages/ollama-nv) are two ways to
  obtain the vectors this store keeps.
- `std.store` in the standard library is an in-memory vector store for
  retrieval-augmented generation. It compares every vector, has no
  metadata filter, no keyword index and no file, and it is the right thing
  for a few thousand documents in one process.
- `std.llm` in the standard library computes embeddings as well as running
  inference.
- `std.json` is the metadata type a document carries and a filter reads.

## Tests

```bash
novo test tests/vsvalue_tests.nv     # 31 tests: documents, filters, fusion, faults
novo test tests/vssearch_tests.nv    # 14 tests: the four plans and what a result says
novo test tests/vsstore_tests.nv     #  8 tests: the trait and the in-memory store
novo test tests/vsio_tests.nv        #  7 tests: the file format and the two dialects
```

The BM25 numbers are Robertson and Walker's `k1` of 1.2 and `b` of 0.75,
and the BM25+ lower bound is Lv and Zhai's. The fusion constant of 60 is
Cormack, Clarke and Buettcher's. The filter semantics are
`vsfilter.matches`, and the two server translations are asserted against it
rather than against a transcript.

The suite asserts what each plan does to `exhaustive`, that a post-filter
is never chosen automatically, and that a score coming out of either server
is a similarity.

The tests compile today and fail at run, each on the
`not implemented: vectorstore-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one at
a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `vsdoc.doc`, `.with_metadata`, `.with_vector`, `.config`, `.with_keyword_index`, `.collection` | no |
| `vsdoc.upsert`, `.upsert_many`, `.delete`, `.compact` | no |
| `vsdoc.get`, `.get_many`, `.doc_count`, `.deleted_count`, `.ids`, `.check_vector` | no |
| `vsfilter.is_clause`, `.all_of`, `.any_of`, `.check`, `.matches` | no |
| `vsfilter.clause_count`, `.fields_of`, `.simplify`, `.is_satisfiable` | no |
| `vsfilter.value_of`, `.value_json` | no |
| `vsbm25.default_params`, `.params`, `.tokenizer`, `.with_stop_words`, `.tokenize` | no |
| `vsbm25.build`, `.empty_index`, `.add`, `.score_all`, `.top_k` | no |
| `vsbm25.idf`, `.doc_frequency`, `.average_length`, `.unknown_terms`, `.term_count` | no |
| `vsrank.default_fusion`, `.fusion`, `.ranking`, `.of_hits`, `.check` | no |
| `vsrank.fuse`, `.contribution`, `.found_in`, `.take_k`, `.overlap` | no |
| `vssearch.plan_for`, `.pre_filter_max`, `.estimate_matches` | no |
| `vssearch.query`, `.text_query`, `.hybrid_query`, `.with_filter`, `.with_plan`, `.with_ef`, `.with_model` | no |
| `vssearch.search`, `.search_docs`, `.exact_search`, `.recall`, `.check`, `.explain` | no |
| `vsstore.memory`, `.with_tokenizer`, `.with_bm25`, `.with_fusion`, `.snapshot` | no |
| `VsStore` for `VsMemory`, for `VsFileStore` and for `VsHttp`: all five methods | no |
| `vsstore.retrieve`, `.ingest`, `.check_query` | no |
| `vsfile.VS_FILE_MAGIC`, `.VS_FILE_VERSION` | yes (they are constants) |
| `vsfile.supported_versions`, `.to_bytes`, `.from_bytes`, `.peek_header`, `.size_bound`, `.is_collection` | no |
| `vsfile.save`, `.load`, `.read_header`, `.is_collection_file`, `.open`, `.flush` | no |
| `vshttp.dialect_name`, `.dialect_of_name`, `.qdrant`, `.chroma`, `.with_header`, `.with_model` | no |
| `vshttp.encode_filter`, `.encode_search`, `.encode_upsert` | no |
| `vshttp.decode_search`, `.decode_get`, `.decode_error` | no |
| `vshttp.search_path`, `.upsert_path`, `.point_id`, `.check_query` | no |
| `vshttp.scroll`, `.health`, `.ensure_collection` | no |
| `vsfault.is_retryable`, `.is_caller_error`, `VsFault.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
