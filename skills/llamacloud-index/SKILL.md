---
name: llamacloud-index
description: This skill should be used when the user wants to "build a Cloud Index", set up a "managed RAG pipeline on LlamaCloud", configure "hybrid search / rerank / metadata filter on LlamaCloud", "ingest documents into a LlamaCloud index", or expose a "managed retrieval endpoint". Owns the end-to-end managed Cloud Index (source → parse/transform → chunk → embed → sink) and retrieval tuning; defers OSS/self-managed RAG to `llamaindex-framework`, raw parsing to `llamacloud-parse`, typed extraction to `llamacloud-extract`, and account/key setup to `llamacloud`.
version: 0.1.0
---

# LlamaCloud Cloud Index — Managed RAG

LlamaCloud **Cloud Index** is a fully managed RAG pipeline: connect a data source, and LlamaCloud
parses, transforms, chunks, embeds, and stores documents into a sink, then exposes a hosted
**retrieval endpoint**. Choose it to get production retrieval without operating ingestion workers
or a vector database. This skill owns the build (source → sink) and retrieval tuning; it does not
teach OSS RAG, raw parsing, or extraction.

## When managed Cloud Index beats self-managed

Prefer Cloud Index when the value is *retrieval that works*, not infrastructure:

- **No infra to run.** Ingestion, embedding, and the vector store are hosted; no workers, no DB ops.
- **Auto-sync sources.** Connected data sources (cloud buckets, drives) re-ingest on a schedule —
  new/changed files flow in without custom pipelines.
- **Parse quality built in.** LlamaParse runs inside the pipeline, so complex PDFs/tables become
  clean chunks without a separate parse step.
- **One retrieval endpoint.** Tuning (hybrid/rerank/filters) is a request parameter, not a redeploy.

Prefer self-managed OSS (`llamaindex-framework` with `VectorStoreIndex` + your own vector DB) when:
you need full control of chunking/embedding code, must keep data on existing local infra, want a
vector store LlamaCloud doesn't host, or are doing rapid offline experimentation. See
`references/ingestion.md` for the full decision table.

## Build sequence (source → sink)

A Cloud Index is defined by an ordered pipeline. Decide each stage deliberately:

1. **Data source** — file upload for one-off corpora; a connected source (cloud bucket, drive) for
   corpora that grow and must stay in sync. Connectors and their exact config are live lookups.
2. **Parse / transform** — toggle LlamaParse and pick a tier. Default (empty params) is the
   cost preset: fast, fine for clean text, weak on complex layout. Set an agentic tier for
   tables/scans/multi-column. Then choose **segmentation** (none / page / element — element gives
   structure-aware units but is incompatible with fast parse) and **chunking** (character / token /
   sentence / semantic). See `references/ingestion.md` for which to pick.
3. **Embedding model** — pick the embedding model (provider key required for hosted providers).
   Optionally add a **sparse model** (`splade` / `bm25` / `auto`) to enable hybrid search at the
   sink. Decide this now: hybrid must be enabled at ingestion, not bolted on at query time.
4. **Sink** — **Fully Managed** sink for the zero-ops path; an external vector store (BYO) when you
   must own the data plane. Hybrid search is only available on sinks that support sparse vectors.
5. **Deploy** — ingestion runs; then copy the retrieval endpoint / API key into the app.

Indexes are created and managed via the **API/SDK** as the durable default. A UI flow also exists,
and the legacy "Index v1" UI is restricted to projects with pre-existing pipelines — treat API/SDK
management as the path to teach.

## Retrieval tuning

The retrieval endpoint accepts tuning per request — no re-ingestion needed (except enabling hybrid,
which is set at build time). Three levers, applied in this order of impact:

- **Retrieval mode** — pick granularity for the query shape: chunk-level for pinpoint fact-finding;
  file-level when the right *document* matters (by metadata, or by holistic content for
  summarization); auto-routed when query types vary. Verify the current mode enum via MCP.
- **Hybrid search** — combine dense (semantic) + sparse (keyword) with an `alpha` blend. Shift
  toward keyword for exact terms/codes/names; toward semantic for paraphrased intent. Requires a
  sparse model configured at build time.
- **Reranking** — retrieve a wide candidate set (high top-k), then rerank and keep the top few.
  Turn on when precision of the final ordering matters more than latency.
- **Metadata filtering** — constrain by file/tag/date/category, or for access control. Use explicit
  filters when the app knows the constraint; use **metadata filter inference** (a schema the system
  reads to pull filters from natural-language queries) when end users phrase constraints in prose.

See `references/retrieval-tuning.md` for the mode-selection table, the hybrid/rerank/filter decision
table, alpha-tuning intuition, and metadata-schema authoring tips.

## Version posture

Teach the **stable Cloud Index** surface. **Cloud Index v2 is beta and deferred** — flag it as beta
and do not make it the default path. Do not present v2 as GA.

## SDK examples (minimal — confirm signatures via MCP)

Use the high-level services SDK. **Confirm the exact package name and method signatures live**
before pinning — the Python package has shifted between `llama-cloud` and `llama-cloud-services`,
and retrieval may run through `llama-index` client classes. These snippets show *shape*, not exact
API:

```python
# Python — query a deployed Cloud Index's retrieval endpoint (illustrative)
# Verify class/param names via the docs MCP before using.
retriever = index.as_retriever(
    dense_similarity_top_k=10,   # widen, then rerank
    alpha=0.5,                   # 0 = keyword, 1 = semantic
    enable_reranking=True,
    rerank_top_n=3,
)
nodes = retriever.retrieve("...query...")
```

```typescript
// TypeScript — shape only; fetch the current @llamaindex client + params via MCP.
const retriever = index.asRetriever({ similarityTopK: 10, enableReranking: true });
const nodes = await retriever.retrieve("...query...");
```

For exact, current creation/retrieval signatures and connector/sink config, use the MCP — do not
copy these verbatim.

## Fetching facts (do not hardcode)

Treat package names/versions, class/method signatures, connector and sink names, parameter schemas,
and credit numbers as **live lookups**. Fetch via:

- `mcp__plugin_llamacloud_docs__search_docs` — concept / BM25 search.
- `mcp__plugin_llamacloud_docs__read_doc` — read a known doc page.
- `mcp__plugin_llamacloud_docs__grep_docs` — exact symbol / regex search.
- **Fallback:** append `index.md` to any `https://developers.llamaindex.ai/...` page URL and `WebFetch` it.

Anchor doc paths:
- `/llamaparse/cloud-index/getting_started/` — build flow, sources, sinks, deploy.
- `/llamaparse/cloud-index/guides/parsing_transformation/` — parse tier, segmentation, chunking, embedding/sparse.
- `/llamaparse/cloud-index/guides/retrieval/advanced/` — hybrid, rerank, metadata filtering & inference.

## Handoffs

- Self-managed / OSS RAG (`VectorStoreIndex`, own vector DB) → `llamaindex-framework`.
- Raw document → clean text/markdown only → `llamacloud-parse`.
- Typed field/JSON extraction → `llamacloud-extract`.
- Account, API key, region, pricing/credits → core `llamacloud`.

## Additional resources

- **`references/ingestion.md`** — managed-vs-self-managed decision table, source choice, parse-tier
  and segmentation/chunking decisions, embedding + sparse-model choices, sink selection, gotchas.
- **`references/retrieval-tuning.md`** — retrieval-mode selection table, hybrid/rerank/metadata-filter
  decision table, alpha intuition, metadata filter-inference schema tips, anti-patterns.
