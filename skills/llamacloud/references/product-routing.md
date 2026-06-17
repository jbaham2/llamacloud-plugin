# Product Routing — full decision tree & boundaries

Durable judgment for choosing among LlamaCloud products and sequencing them. Facts (signatures,
limits) are fetched live; this file is *which product, when, and why*.

## Decision tree

```
What do you have / want?
│
├─ A document, want clean text/markdown for my OWN LLM step
│     → Parse                                   (llamacloud-parse)
│
├─ A document, want TYPED fields matching a schema
│     → Extract                                 (llamacloud-extract)
│       (Extract parses internally — don't pre-Parse unless you also need raw text)
│
├─ Many docs, want semantic search / RAG / Q&A over them
│     ├─ want it managed (hosted ingest + retrieval, no infra)
│     │     → Cloud Index                       (llamacloud-index)
│     └─ want full control / local / custom vector DB
│           → OSS framework                      (llamaindex-framework)
│
├─ Mixed pile of docs arriving together
│     ├─ need to know each doc's TYPE / route it → Classify   (llamacloud-classify)
│     └─ one file is several docs concatenated   → Split      (llamacloud-split)
│
├─ Messy spreadsheet, want rows/tables I can reason over
│     → Sheets                                  (llamacloud-sheets)
│
└─ Need to parse OFFLINE / no API key / OSS-only
      → LiteParse                               (liteparse)
```

## Key boundary calls

### Parse vs Extract
- **Extract** when the deliverable is structured data (JSON/Pydantic objects). It runs its own
  parsing pass; manually chaining Parse→your-own-LLM→fields reinvents Extract and usually loses
  citation/grounding support.
- **Parse** when the deliverable is text/markdown you will feed to *your own* prompt, index, or
  pipeline, or when you need the raw layout-faithful content (tables, reading order) and will do
  the structuring yourself.
- Both: only when you genuinely need the raw markdown *and* typed fields from the same run.

### Cloud Index (managed) vs llamaindex-framework (OSS)
Choose **Cloud Index** when: you want hosted ingestion + a managed vector store + a retrieval
endpoint without operating infra; the team is small or RAG isn't the core IP; you want the console
to manage data sources/sinks.
Choose **the OSS framework** when: you need a specific vector DB or embedding model, full control of
chunking/retrieval, local/air-gapped operation, or RAG is core IP you want to own end to end.
Cloud Index v2 is **beta** — build on the stable Cloud Index surface and flag v2 explicitly.

### Classify vs Split
- **Classify** answers "what *kind* of document is this?" → routes each doc to the right pipeline.
- **Split** answers "where does one logical document end and the next begin?" inside a single
  concatenated file → emits sub-documents.
- Common combo: Split a merged scan into sub-docs, Classify each, then Parse/Extract per type.

### LlamaParse vs LiteParse
- **LlamaParse** (cloud, needs key): highest quality, agentic/LLM-backed parsing, tier control,
  scale. Default for production document quality.
- **LiteParse** (local, no key, OSS): fast, free, offline, no data leaves the machine. Choose for
  privacy/air-gapped needs, dev loops, simple/clean documents, or cost-sensitive bulk where quality
  tolerance is loose.

## Multi-product pipelines (sequence them here, then hand off)

- **Invoice/contract intake:** (Split if batched) → Classify (type) → Extract (per-type schema).
- **RAG over a doc corpus:** Parse (or let Cloud Index parse) → chunk/embed → Cloud Index retrieval.
- **Spreadsheet analytics:** Sheets → downstream reasoning / Extract for typed cells.
- **Mixed inbox:** Classify first to fan documents to the correct per-type sub-pipeline.

## Anti-patterns

- Chaining Parse → hand-rolled extraction prompt when Extract would give schema-validated,
  cited output in one call.
- Standing up a self-managed vector DB for a small RAG MVP when Cloud Index would ship faster.
- Using cloud LlamaParse for air-gapped/private data where LiteParse satisfies the quality bar.
- Treating Classify/Split/Sheets (beta) as load-bearing for a strict SLA without a fallback.
- Defaulting to `/v1/` Parse/Extract code paths — v2 is the GA surface.
