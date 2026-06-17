# Component map: category → job, plus selection judgment

Which component category does which job, and how to choose within each. Decision tables
only — fetch exact class/method names and parameters live (SKILL.md "Fetching facts").

## Category → job (the full map)

| Stage | Category | Job | When you touch it |
|-------|----------|-----|-------------------|
| Load | Readers / connectors | Pull source data into `Document`s | Always |
| Load | Node parsers / splitters | Chunk `Document`s into `Node`s | Always (tune the chunking) |
| Index | Indexes | Organize nodes for retrieval | Always (pick the type) |
| Index | Embeddings | Encode text → vectors | Whenever using a vector index |
| Index/Query | LLMs | Generate text (synthesis, extraction, reasoning) | Always |
| Store | Storage context (doc/index/vector store) | Persist; swap backends | When prototyping ends |
| Query | Retrievers | Fetch candidate nodes | Always (tune `top_k`) |
| Query | Node postprocessors | Filter / rerank / cutoff | When retrieval is noisy |
| Query | Response synthesizers | Compose answer from nodes | Always (pick the mode) |
| Query | Routers | Pick among retrievers/engines | Multi-source or mixed-intent |
| Query | Query / chat engines, agents | Orchestrate the flow | Always (pick the interface) |
| Config | Settings | Set default LLM + embed model globally | Once, at startup |

## Choosing an index type

| Index | Retrieval shape | Use when | Avoid when |
|-------|-----------------|----------|------------|
| **Vector store index** | Top-k by embedding similarity | Default. Semantic Q&A over a corpus. | (Rarely the wrong choice) |
| **Summary index** | Reads all nodes, synthesizes | "Summarize / answer over the whole document set" | Large corpora — reads everything (cost) |
| **Property graph index** | Graph traversal + vector | Multi-hop, relationship, "how is X connected to Y" | Plain factoid lookup (overkill) |
| **Keyword table index** | Exact keyword → node map | Precise term/code/ID lookup | Paraphrased / semantic queries (misses) |
| **Document summary index** | Retrieve doc by its summary, then nodes | Many docs where doc-level relevance matters first | Single small corpus |

Default to vector. Combine indexes behind a **router** only when one corpus genuinely has
two query shapes (e.g. semantic + exact-ID). Do not multiply indexes speculatively.

## Choosing a store

| Option | Use when | Watch out for |
|--------|----------|---------------|
| In-memory default + persist/load to disk | Prototype, single process, fits in memory | Forgetting to persist → silent re-embed every run |
| External vector database | Data > memory, shared across processes, prod durability, scale filtering | Operational ownership; embedding-model consistency across writes |

Trigger to migrate: data outgrows memory, multiple services share the index, concurrent
writes, or production durability. Migrating means re-indexing into the new store. For a
team that does not want to operate a store at all, the managed option is `llamacloud-index`
— not a bigger self-hosted store.

## Choosing a retrieval refinement

| Add this | When | Skip when |
|----------|------|-----------|
| Higher `top_k` | Answer evidence is being missed | Synthesizer is already noisy/expensive |
| Reranker (postprocessor) | Retrieved nodes are roughly-relevant but unranked; need precision from a wide net | Plain vector retrieval already returns clean top results |
| Similarity cutoff | Weak/irrelevant matches leak into the answer | Recall is the problem, not precision |
| Metadata filter | Queries scope to a subset (date, source, tenant) | No usable metadata was attached at load |

Order of operations: tune `top_k` first (free), add cutoff/filter next, add a reranker
last. A reranker is a precision tool, not a default.

## Choosing a response synthesizer mode

Mode controls how retrieved chunks become an answer — trading LLM calls (latency + cost)
against completeness.

| Mode | LLM calls | Cost / latency | Use for |
|------|-----------|----------------|---------|
| **compact** | Few (chunks packed per prompt) | Low | Default. Normal Q&A. |
| **refine** | One per chunk, sequential | Highest | Maximum-detail answers; small node counts |
| **tree_summarize** | Logarithmic (tree) | Medium | Summarizing across many chunks |
| **simple_summarize** | One (truncates to fit) | Lowest | Fast, lossy summaries; speed over detail |
| **accumulate** | One per chunk | High | Same question against each chunk independently |
| **no_text** | None | Minimal | Inspect retrieved nodes without generating |

Stay on `compact` until an answer is provably incomplete (then `refine`/`tree_summarize`)
or provably too slow/expensive (then `simple_summarize`).

## Choosing the query interface

| Interface | Use when | Defer to |
|-----------|----------|----------|
| Query engine | One-shot Q&A, no memory | — |
| Chat engine | Follow-ups reference earlier turns | — |
| Agent | Answering needs tool use / multi-step reasoning | `llamaindex-workflows` for event-driven orchestration |

## Cross-stage consistency rules (easy to violate)

- The embedding model used to query MUST match the one used to index. A mismatch silently
  tanks recall. Changing it requires re-indexing.
- Metadata that the query filters on must be attached at load time. It cannot be filtered
  on if it was never stored.
- Chunk size affects every downstream stage. Changing it requires re-indexing, so settle
  it early.
