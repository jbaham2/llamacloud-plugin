---
name: llamaindex-framework
description: This skill should be used when the user wants to "build a RAG pipeline with LlamaIndex (OSS)", work with "VectorStoreIndex / query engine", follow the "load-index-store-query" model, consult "LlamaIndex component/module guides", or do "self-managed retrieval in code". Owns the open-source LlamaIndex Python/TS framework mental model and component selection; defers managed/hosted RAG to llamacloud-index, agent orchestration to llamaindex-workflows, deployment to llamaindex-deploy, and cloud parsing to llamacloud-parse.
version: 0.1.0
---

# LlamaIndex Framework (OSS) — Build RAG Yourself

Guide building retrieval-augmented generation in code with the open-source LlamaIndex
framework: own the data flow, pick components, run retrieval and synthesis in your own
process. Use this skill for self-managed RAG. For a fully managed pipeline (hosted
indexing, parsing, retrieval API) hand off to `llamacloud-index`.

## The mental model: load → index → store → query

Every LlamaIndex RAG application is the same four-stage pipeline. Hold this map; it
determines which component category does each job.

1. **Load** — get data in and chunk it. Readers/connectors ingest from files, URLs,
   databases, APIs. A node parser splits documents into `Node`s (chunks + metadata).
2. **Index** — build a queryable structure. Most commonly embed nodes into a
   `VectorStoreIndex`. Embeddings turn text into vectors for semantic similarity.
3. **Store** — persist the index so re-indexing is not repeated on every run. Either
   persist the default in-memory stores to disk, or back the index with an external
   vector database.
4. **Query** — answer questions. A query engine wires a retriever (fetch top-k nodes) →
   optional node postprocessors (filter/rerank) → response synthesizer (LLM composes the
   answer). Add memory with a chat engine; add reasoning/tools with an agent.

A fifth concern, **evaluate**, measures retrieval and answer quality once the pipeline
runs. Treat it as the feedback loop, not a build step. When an answer is wrong, the
mental model tells you where to look: evaluate retrieval and generation separately, then
fix the stage that broke (chunking, embedding, `top_k`, or synthesizer) rather than
guessing at the prompt.

This skill owns the **framework-vs-managed decision** too. Build with the OSS framework
when the team wants control over chunking, the vector store, retrieval logic, and
hosting, and is willing to operate them. Choose the managed Cloud Index
(`llamacloud-index`) when the team would rather not run indexing infrastructure at all —
hand off the whole pipeline. The two are not mutually exclusive at the component level,
but pick one as the spine of the application.

The framework offers a **high-level path** (`VectorStoreIndex.from_documents(...)` then
`.as_query_engine()` — sensible defaults, few lines) and a **low-level path** (assemble
retriever, postprocessors, and synthesizer by hand). Start high-level: get an answer end
to end, measure it, and only then drop to the low-level API for the one stage that needs
control. Reaching for the low-level API, custom retrievers, routers, or sub-question
decomposition before the defaults have been measured is the most common way these
pipelines accrue complexity that never pays for itself. See `references/rag-pipeline.md`
for the sequence and the key decision at each stage.

## Component map (which category for which job)

| Stage | Category | Job |
|-------|----------|-----|
| Load | Readers / connectors | Ingest source data into `Document`s |
| Load | Node parsers / text splitters | Chunk documents into `Node`s |
| Index | Indexes | Organize nodes for retrieval (vector, summary, keyword, property graph) |
| Index | Embeddings | Encode text to vectors |
| Index/Query | LLMs | Generate text (extraction, synthesis, agent reasoning) |
| Store | Storage context | Doc store + index store + vector store; persist vs re-index |
| Query | Retrievers | Fetch candidate nodes for a query |
| Query | Node postprocessors | Filter, rerank, apply similarity cutoffs |
| Query | Response synthesizers | Compose the final answer from nodes |
| Query | Routers / query engines / chat engines / agents | Route and orchestrate the query flow |

Selection judgment — which index, which retrieval shape, which synthesizer mode, when to
move from local to an external vector store — lives in `references/component-map.md`.
Consult it before defaulting; the defaults are good but not universal.

## Key choices, fast

- **Index type:** `VectorStoreIndex` for semantic search (the default and the right
  answer for most RAG). Summary index for "read everything and synthesize." Property
  graph index for relationship/multi-hop questions. Keyword table for exact-term lookup.
- **Store:** prototype with the in-memory default and `persist`/`load` to disk. Move to
  an external vector database when data outgrows memory, multiple processes share the
  index, or production durability matters.
- **Synthesizer mode:** `compact` (default) for normal Q&A; `tree_summarize` for
  summarizing many chunks; `refine` for maximum detail at higher cost. Mode trades LLM
  calls (latency/cost) against answer completeness — table in `component-map.md`.
- **Query interface:** query engine for one-shot Q&A; chat engine when conversation
  memory matters; agent when the query needs tool use or multi-step reasoning. For full
  event-driven agent orchestration, defer to `llamaindex-workflows`.

## Packaging note (durable)

LlamaIndex is modular: a small core plus many independently versioned integration
packages (each reader, vector store, LLM, and embedding provider ships separately).
Install the core plus only the integrations a pipeline uses. The exact distribution
names have shifted over time — confirm them live (see below) rather than hardcoding from
memory. Illustrative Python uses the `llama-index` family; TypeScript uses LlamaIndex.TS.

```python
# Python — high-level, illustrative (confirm package/signature via docs)
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
docs = SimpleDirectoryReader("./data").load_data()
index = VectorStoreIndex.from_documents(docs)
answer = index.as_query_engine().query("What changed in Q3?")
```

```typescript
// TypeScript (LlamaIndex.TS) — illustrative
import { VectorStoreIndex, Document } from "llamaindex";
const index = await VectorStoreIndex.fromDocuments([new Document({ text })]);
const res = await index.asQueryEngine().query({ query: "What changed in Q3?" });
```

Set the embedding model and LLM once via the global `Settings` object (or pass them
per-call) instead of threading them through every component.

## Fetching facts (signatures, package names, parameters)

Do not trust memory for exact class/method signatures, parameter names, or current
package names — they drift. Look them up live:

- `mcp__plugin_llamacloud_docs__search_docs` — concept / BM25 search.
- `mcp__plugin_llamacloud_docs__read_doc` — read a known doc page.
- `mcp__plugin_llamacloud_docs__grep_docs` — exact symbol / regex search.
- **Fallback (verified working for OSS framework docs):** append `index.md` to any
  `https://developers.llamaindex.ai/...` page URL and WebFetch it. Prefer this fallback
  for OSS `framework/` pages.

Anchor doc paths for this pillar:
- `https://developers.llamaindex.ai/python/framework/understanding/rag/` — the staged model.
- `https://developers.llamaindex.ai/python/framework/module_guides/` — component index.
- `.../module_guides/indexing/index_guide/` — index type comparison.
- `.../module_guides/storing/` — persistence and stores.
- `.../module_guides/querying/response_synthesizers/` — synthesizer modes.

## Boundaries (defer, do not teach here)

- **Managed / hosted RAG** (LlamaCloud indexing, retrieval API, no infra) →
  `llamacloud-index`. Choose managed over this framework when the team does not want to
  run/maintain the indexing and vector store themselves. (Cloud Index v2 is beta; teach
  stable Cloud Index there.)
- **Event-driven agent orchestration** (multi-step `Workflow`, steps/events) →
  `llamaindex-workflows`.
- **Deploying agents/services** → `llamaindex-deploy`.
- **Cloud document parsing** (complex PDFs, tables, OCR) → `llamacloud-parse`. This skill
  uses basic local readers; reach for managed parsing when documents are hard.
