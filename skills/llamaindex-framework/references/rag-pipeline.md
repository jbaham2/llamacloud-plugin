# RAG pipeline: the sequence and the decision at each stage

The load → index → store → query sequence, with the one decision that matters most at
each stage and the failure it prevents. Pull exact signatures from the docs (see SKILL.md
"Fetching facts"); this file is judgment, not API reference.

## Stage 1 — Load

**Job:** turn source data into `Document`s, then split into `Node`s.

**Key decision: chunk size and splitting strategy.** This is the highest-leverage,
most-overlooked choice in the whole pipeline. Chunks that are too large bury the answer
in noise and waste context; too small lose the surrounding meaning needed to answer.

- Default sentence-style splitting with overlap works for prose. Tune chunk size to the
  content's natural unit (a paragraph, a function, a table row).
- Use structure-aware parsers (markdown, code, JSON) when the source has structure worth
  preserving — splitting on structure beats splitting on character count.
- Attach metadata at load time (source, section, date). Metadata enables filtering at
  query time and is far cheaper to add now than to backfill.

**Failure prevented:** "retrieval returns the right document but the wrong span." Almost
always a chunking problem, not a retrieval problem.

**Reader choice:** local readers (directory, web, simple file types) are fine for clean
inputs. For complex PDFs, scanned docs, dense tables, or OCR, defer to managed parsing
(`llamacloud-parse`) — bad parse in, bad answers out, and no downstream tuning fixes it.

## Stage 2 — Index

**Job:** build the structure queries run against.

**Key decision: which index type** (full table in `component-map.md`). For ~95% of RAG,
`VectorStoreIndex` is correct. Reach for another only when the question shape demands it
(summary index for whole-corpus synthesis, property graph for multi-hop/relationship
queries, keyword table for exact-term lookup).

**Embedding model decision:** the embedding model determines retrieval quality more than
any query-time knob. Match the model to the domain and language; keep the *same* model for
indexing and querying — a mismatch silently destroys recall. Changing the embedding model
means re-indexing.

**Failure prevented:** "good chunks exist but never surface." Usually a weak or mismatched
embedding model, or the wrong index type for the question.

## Stage 3 — Store

**Job:** persist so indexing is paid once, not every run.

**Key decision: in-memory + persist-to-disk vs external vector database.**

- **Default (in-memory store, persist/load to disk):** correct for prototyping, single
  process, datasets that fit in memory. Persist after building; load on startup. The
  trap is *forgetting to persist* and silently re-embedding (cost + latency) every run.
- **External vector database:** move here when data outgrows memory, multiple processes
  or services share one index, you need concurrent writes, or production durability and
  filtering at scale matter. Many external stores hold both vectors and node content, so
  a separate doc/index store is often unnecessary.

The storage context has three parts — doc store (nodes), index store (index metadata),
vector store (embeddings). Know they exist so persistence and store-swapping make sense.

**Failure prevented:** "startup is slow and the embedding bill is huge" — re-indexing
that should have been a load. And "data lost on restart" — never persisted at all.

## Stage 4 — Query

**Job:** retrieve, refine, synthesize an answer.

The query engine composes three steps; tune them in order of impact:

1. **Retriever — `top_k`.** How many nodes to fetch. Too low misses evidence; too high
   floods the synthesizer with noise and cost. Tune this first; it is the cheapest lever.
2. **Node postprocessors (optional) — rerank and cutoff.** Add a reranker when retrieval
   pulls roughly-relevant-but-noisy nodes: retrieve a wider `top_k`, then rerank down to
   the best few. Add a similarity cutoff to drop weak matches. Skip until retrieval
   quality is actually the bottleneck — do not add a reranker reflexively.
3. **Response synthesizer — mode.** `compact` (default) for normal Q&A; `tree_summarize`
   when synthesizing across many chunks; `refine` for maximum detail at higher LLM cost.
   Mode trades LLM calls (latency/cost) against completeness — table in `component-map.md`.

**Interface choice:**
- **Query engine** — stateless one-shot Q&A.
- **Chat engine** — adds conversation memory; use when follow-ups reference prior turns.
- **Agent** — use when answering needs tool calls or multi-step reasoning over data. For
  full event-driven orchestration of multi-step agents, defer to `llamaindex-workflows`.

**Failure prevented:** "the model hallucinates / ignores the context." Usually retrieval
fed it the wrong or too-few nodes (fix stages 1–2 and `top_k`), not a prompt problem.

## Stage 5 — Evaluate (the loop, not a step)

Measure retrieval (did the right nodes come back?) separately from generation (was the
answer faithful and correct?). When quality is poor, this split tells you *which stage*
to fix instead of guessing. Common path: bad answer → check retrieval → if retrieval is
fine, fix synthesizer/prompt; if retrieval is bad, fix chunking or the embedding model.

## Anti-patterns

- Jumping to the low-level API before the high-level defaults have been measured.
- Adding a reranker, query routing, or sub-question decomposition before confirming plain
  vector retrieval is the bottleneck — complexity that does not pay for itself.
- Re-embedding on every run because the index was never persisted.
- Changing the embedding model without re-indexing (indexing/query model mismatch).
- Forcing a non-vector index because it sounds sophisticated; vector + good chunks wins.
