# Ingestion: source → sink build decisions

Durable judgment for designing the managed pipeline. Exact connector names, sink names, model IDs,
parameter schemas, and credit numbers are live lookups (MCP / `index.md` + WebFetch) — not here.

## Managed Cloud Index vs self-managed OSS

| Signal | Managed Cloud Index | Self-managed (`llamaindex-framework`) |
|---|---|---|
| Ops appetite | Want zero infra (no workers, no vector DB) | Willing to run/scale your own stack |
| Source freshness | Corpus grows; want scheduled auto-sync | One-shot or you control re-ingestion |
| Parse quality | Want LlamaParse inline on complex docs | Already have a parsing/cleaning step |
| Vector store | A hosted/managed sink (or supported BYO) is fine | Need a specific store LlamaCloud doesn't host |
| Chunking/embedding control | Defaults + a few knobs are enough | Need custom code in the transform path |
| Data locality | Cloud (or chosen region) is acceptable | Must stay on existing local/on-prem infra |
| Iteration mode | Production retrieval that "just works" | Rapid offline experimentation in code |

Rule of thumb: managed wins when the goal is *retrieval that works in production*; OSS wins when the
goal is *control over the pipeline internals*. Mixed corpora that must be classified/split first are
upstream concerns — sequence those before the index (see core `llamacloud`).

## Data source

- **File upload** — one-off or static corpora; simplest, no credentials.
- **Connected source** (cloud bucket / drive / store) — corpora that change; the index re-ingests on
  a schedule so new and updated files appear without custom code. This auto-sync is the main reason
  to prefer a connector over manual upload. Confirm the available connectors and their auth via MCP.

Gotcha: deletions and updates depend on the connector's sync semantics — verify how a source handles
removed files before relying on the index to prune stale content.

## Parse / transform

Parsing runs inside the pipeline (LlamaParse). The decision is **tier vs cost**, then structure.

| Document profile | Parse tier | Why |
|---|---|---|
| Clean digital text, simple layout | Default / cost preset | Cheapest, fast; layout doesn't matter |
| Tables, multi-column, forms, scans | Agentic tier | Layout fidelity drives chunk quality |
| Mixed corpus | Agentic for the hard docs | Bad parsing poisons every downstream chunk |

Parse quality is upstream of everything: a table flattened into garbage text cannot be fixed by
chunking or reranking. When retrieval is weak on structured docs, suspect the parse tier first.

### Segmentation (how the document is divided before chunking)

| Mode | Use when | Caveat |
|---|---|---|
| None | Let chunking handle everything | Loses page/element boundaries |
| Page | Page boundaries are meaningful (e.g. citations) | — |
| Element | Want title/paragraph/list/table-aware units | Incompatible with fast parse mode |

Element segmentation gives the most structure-aware retrieval but forces a non-fast parse tier —
budget for it.

### Chunking

| Mode | Use when |
|---|---|
| Character | Crude size cap; rarely the best choice |
| Token | Need to respect an LLM context window precisely |
| Sentence | Want clean linguistic boundaries, paragraph-aware |
| Semantic | Want context-coherent chunks; best recall, higher cost |

Default to token or sentence; reach for semantic when chunks keep splitting mid-thought and hurting
recall. Tune `chunk_size` / `chunk_overlap` (exact params via MCP): larger chunks = more context per
hit but coarser matching; more overlap = fewer boundary misses at higher storage/credit cost.

## Embedding + sparse model

- **Embedding model** — pick per quality/cost/region; hosted providers need a provider key. Confirm
  the supported model list live; do not hardcode model IDs.
- **Sparse model** (`splade` / `bm25` / `auto`) — adds keyword vectors so the sink can do **hybrid
  search**. Decide at build time: hybrid cannot be enabled later without re-ingestion. Add it when
  queries contain exact terms, codes, names, or IDs that pure semantic search blurs.

## Sink

- **Fully Managed sink** — zero-ops default; hosts the vectors for you.
- **External / BYO vector store** — when you must own the data plane or already run a supported store.

Hybrid search requires a sink that stores sparse vectors. If hybrid is a requirement, confirm the
chosen sink supports it *before* deploying — otherwise the sparse model is wasted.

## Build-order gotchas

- **Hybrid is a build-time decision**, not a query-time toggle — configure the sparse model and a
  hybrid-capable sink up front.
- **Re-ingestion cost** — changing parse tier, segmentation, chunking, or embedding model means
  re-processing the corpus (credits + time). Get these right before deploying a large corpus; test
  on a sample first.
- **Parse tier dominates quality** — spend the credits on parsing structured docs; it's cheaper than
  debugging bad retrieval later.
- **Cloud Index v2 is beta/deferred** — build on the stable surface; don't default to v2.
