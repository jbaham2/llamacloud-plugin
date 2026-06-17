# Retrieval tuning: modes, hybrid, rerank, metadata filters

Durable judgment for tuning the managed retrieval endpoint. Parameter names and exact schemas drift
— confirm via MCP (`mcp__plugin_llamacloud_docs__search_docs` / `read_doc` / `grep_docs`) or the
`index.md` + WebFetch fallback. Anchor: `/llamaparse/cloud-index/guides/retrieval/advanced/`.

All tuning below is **per-request** and needs no re-ingestion — except hybrid search, which requires
a sparse model configured at build time (see `ingestion.md`).

## Retrieval mode — pick granularity for the query shape

Four modes (verify the current enum live before quoting names):

| Mode | Use when | Returns |
|---|---|---|
| `chunks` | Fact-finding about a specific section; pinpoint answers from a large corpus | Best-matching chunks |
| `files_via_metadata` | Query targets a *file* identifiable by its metadata (name, tag, attribute) | Whole files via metadata match |
| `files_via_content` | Holistic/summarization query where the relevant file isn't known up front | Files chosen by content |
| `auto_routed` | Query types vary widely; let the system route to the right strategy | System's choice |

Default to `chunks` for typical RAG question-answering. Move to a file-level mode when the unit of
truth is a document (a contract, a report) rather than a passage. Use `auto_routed` when one endpoint
serves heterogeneous queries and you don't want to classify them yourself — at the cost of less
predictable behavior.

## Tuning levers — decision table

| Lever | Turn on / shift when | Cost / caveat |
|---|---|---|
| **Hybrid search** | Queries contain exact terms, codes, names, IDs, acronyms | Needs sparse model at build time; only on hybrid-capable sinks |
| **`alpha` blend** | Toward keyword (low) for literal matching; toward semantic (high) for paraphrased intent | 0.5 is balanced; one value won't suit every query |
| **Reranking** | Final-ordering precision matters; top results must be the *best*, not just close | Extra latency + cost per query |
| **High top-k** | Pair with reranking — retrieve wide, rerank down | Wasteful without a reranker; more candidates ≠ better order |
| **Metadata filtering** | App knows the constraint (file/date/category/tenant) or needs access control | Filter keys must exist in indexed metadata |
| **Filter inference** | End users express constraints in natural language | Needs a well-described metadata schema; adds an inference step |

### Hybrid + alpha intuition

Dense (semantic) search blurs exact tokens; sparse (keyword) search nails them but misses paraphrase.
Hybrid blends both via `alpha` (0 = pure keyword, 1 = pure semantic). Symptoms and fixes:

- Missing rows with exact part numbers / legal cites / names → enable hybrid, shift `alpha` toward
  keyword.
- Missing obviously-relevant paraphrased passages → shift `alpha` toward semantic.
- No single `alpha` works → the corpus mixes literal and conceptual queries; consider `auto_routed`
  or per-query alpha rather than one global value.

### Reranking pattern

The high-value pattern is **retrieve wide, rerank narrow**: set a generous `dense_similarity_top_k`,
enable reranking, return a small `rerank_top_n`. This recovers relevant chunks the first-stage
retriever ranked low. Skip reranking when latency is tight and first-stage ranking is already good
enough — it adds a model call per query.

## Metadata filtering

- **Explicit filters** — `(key, operator, value)` predicates the application supplies. Use for tenant
  isolation, access control (user/group IDs), date ranges, and category scoping. The filter keys must
  have been written into document metadata at ingestion.
- **Metadata filter inference** — define a schema (Pydantic-style model) describing the metadata
  fields; the system reads natural-language queries and extracts filters automatically. Use when end
  users phrase constraints in prose ("financial reports from 2023") rather than the app knowing them.

Schema authoring tips for good inference:
- Write clear docstrings and per-field descriptions — the model relies on them.
- Use granular type annotations; `date` / `datetime` types with format notes for dates.
- Use `Enum` for constrained string values, and specify required casing.
- Keep the schema to fields that actually exist in the indexed metadata.

## Anti-patterns

- **Expecting hybrid at query time without building for it** — sparse model and a compatible sink are
  build-time prerequisites.
- **Cranking top-k without a reranker** — more candidates don't improve ordering; they just add noise
  and cost.
- **Filtering on un-indexed keys** — a filter on metadata that was never written returns nothing.
  Verify the field exists at ingestion.
- **One global `alpha` for a mixed query workload** — route or vary per query instead.
- **Tuning retrieval to fix bad parsing** — if structured docs retrieve poorly, fix the parse tier
  (see `ingestion.md`); no `alpha` or reranker compensates for garbled chunks.
