# Recipe Finder — search method, scenario→category map, fit rubric

Durable judgment for routing a scenario to an existing recipe. No stale facts (exact URLs and
notebook filenames live in the catalogs; fetch them). This file holds the *how to search*, the
*where it lives*, and the *does it fit* judgment.

## Search method (detailed)

Goal: surface the 1–3 recipes that genuinely match, plus an honest gap note. Order of operations:

1. **Decompose the scenario** into three axes:
   - *Data type* — PDF, scanned image, spreadsheet, merged/bundled file, HTML, audio, plain text.
   - *Product / layer* — raw parse, typed extract, split, sheets, managed index/RAG, or pure
     LlamaIndex framework (agents, query engines, integrations).
   - *Pattern* — one-shot parse, schema-driven extraction, retrieval, agent/workflow, evaluation,
     specific integration (a named vector store / LLM / embedding model).
2. **Pick the catalog** from the axes:
   - LlamaCloud *product* scenario (parse/extract/index/split/sheets) → **Cookbooks**.
   - Framework *pattern* or third-party *integration* → **examples Gallery**.
   - Mixed (e.g. "parse PDFs then build agentic RAG") → both; cite a cookbook for the parse/extract
     leg and a gallery notebook for the agent/RAG leg.
3. **Query the live catalog** (tools in SKILL.md "Fetching facts"):
   - First pass: `search_docs` with *scenario noun + pattern* ("invoice extraction citations",
     "agentic rag workflow", "hybrid search qdrant").
   - If you know the class/engine/integration name, `grep_docs` it exactly
     (e.g. `SubQuestionQueryEngine`, `CitationQueryEngine`, `LlamaCloudIndex`, a vendor name).
   - Read the top hit with `read_doc` to confirm it does what the user needs.
4. **Cross-check the hub index pages.** Fetch `…/llamaparse/cookbooks/index.md` and
   `…/python/examples/index.md` for the *currently curated* entry points and fresh URLs. The hub is
   the source of truth for "what's official right now"; the sidebar long-tail is broader but noisier.
5. **Return** topic + URL + one line of why, ranked by fit. Note coverage gaps explicitly.

## Scenario → category map (fuller)

Start here, then search/fetch for the exact current notebook. Categories, not promises of a specific
file.

### Cookbooks (LlamaCloud products)

| Scenario | Subdir / anchor | Notes |
| --- | --- | --- |
| First parse / hello-world parse | parse — Starter | basic demo |
| Multimodal parse (images + text, complex layout) | parse — Advanced | multimodal demo |
| Inspect the parse result object / JSON structure | parse | json / json_tour |
| Tables, charts, financial layout | parse | table_extraction, get_charts |
| Scanned / multilingual / non-English | parse | languages demo |
| Select specific pages only | parse | selected_pages demo |
| Domain doc (insurance, contracts) | parse | insurance demo |
| Parse → load into a DB (e.g. MongoDB) | parse / misc | mongodb demo |
| Batch-parse a folder at scale | parse / batch | batch script |
| First extraction agent / resume screening | extract — Starter | resume_screening |
| Financial / fund / asset report extraction | extract — Advanced | asset_manager_fund_analysis |
| Index + search + chat over data (managed) | index — Starter | getting_started |
| Messy Excel / CSV → normalized rows | sheets | beta |
| Split one merged file into sub-documents | split | beta |

### Gallery (LlamaIndex framework)

| Scenario | Curated hub section | If not in hub, search the sidebar for… |
| --- | --- | --- |
| ReAct / function-calling / code-act / multi-agent | Agents | the agent type by name |
| RAG-as-a-workflow, agent-from-scratch, text-to-SQL | Agentic Workflows | "workflow", "text to sql" |
| Pick a vector store (Qdrant, Pinecone, pgvector, Weaviate, Milvus, Chroma, Redis, MongoDB Atlas) | Vector Stores | the vendor name |
| Pick an embedding model (OpenAI, Cohere, Voyage, Jina, HF, Ollama) | Embedding Models | the vendor name |
| Use a specific LLM (Anthropic, Bedrock, Gemini/Vertex, Mistral, Ollama) | LLM Integrations | the provider name |
| Query engines (sub-question, citation, router, SQL) | — | "query engine" + variant |
| Chat engines / memory | — | "chat engine", "memory" |
| Retrievers / node postprocessors / rerankers | — | "retriever", "rerank", "postprocessor" |
| Structured / typed outputs from the framework | — | "structured output", "pydantic program" |
| Evaluation (faithfulness, relevancy, correctness) | — | "evaluation" + metric |
| Observability / tracing | — | "observability", a tracing vendor name |
| Finetuning (embeddings, rerankers, LLMs) | — | "finetuning" + target |
| Knowledge / property graph | — | "property graph", "knowledge graph" |
| Ingestion pipeline / node parsing / chunking | — | "ingestion pipeline", "node parser" |
| Multimodal (image/audio) RAG | — | "multimodal" |
| Managed indexes (LlamaCloud) inside the framework | — | "LlamaCloudIndex", "managed index" |

The hub `index.md` only exposes a curated subset of sections; the long-tail categories above live in
the gallery sidebar — reach them via `search_docs`/`grep_docs` rather than expecting them on the hub.

## Fit rubric — does the recipe actually match?

A keyword hit is not a fit. Before recommending, check:

1. **Data type matches.** A "parse a PDF" cookbook does not cover a scanned image or a spreadsheet —
   route those to the scanned/multilingual parse demo or to sheets respectively.
2. **Product/layer matches.** "Extract structured fields" ≠ "parse to markdown" ≠ "build a RAG
   index." Confirm the recipe operates at the layer the user asked about.
3. **Pattern matches.** A basic single-shot RAG notebook does not satisfy "agentic RAG with
   citations and routing." Prefer the workflow/agentic example, or note the gap.
4. **Integration matches.** If the user named a vendor (Qdrant, Anthropic, Voyage), the recipe must
   use *that* vendor, not a generic stand-in — unless the user only wants the pattern.
5. **Currency.** If `read_doc`/WebFetch shows the notebook is deprecated or the repo is sunset,
   flag it and prefer the supported equivalent. (The `llama_cloud_services` examples repo carries a
   maintenance-end notice — verify before recommending and prefer current docs.)

## How to present the result

- Lead with the single best match: topic, URL, one line of why it fits.
- Add up to two alternates only if they cover a different facet.
- State coverage gaps plainly ("no recipe combines X and Y; closest covers X, add Y via …").
- Hand off implementation to the product skill — do not start teaching the recipe here.
- When nothing fits, say so and route to the nearest product skill to build from scratch. An honest
  "no direct recipe" beats a forced weak match.
