---
name: llamaindex-cookbooks
description: This skill should be used when the user asks "is there an example/cookbook/notebook for X", wants to "find a recipe/sample for this scenario", or says "show me a LlamaIndex example for …" — routing a described scenario to the most relevant LlamaParse cookbooks and LlamaIndex examples-gallery notebooks. Owns scenario→recipe discovery and coverage checking; defers actually teaching parse/extract/index/split/sheets to the respective product skills.
version: 0.1.0
---

# LlamaIndex Cookbooks & Examples Finder

Route a user's described scenario to the most relevant existing recipes: the
**LlamaParse/LlamaCloud cookbooks** and the **LlamaIndex examples gallery**. This skill is a
*router into the examples* — it surfaces topics and URLs for inspiration and coverage checking. It
does NOT re-teach any product. Once a recipe is found, hand the actual how-to back to the matching
product skill.

## When this skill runs

Trigger on discovery questions, not how-to questions:

- "Is there a cookbook/notebook/example for <scenario>?"
- "Find me a recipe/sample that does <thing>."
- "Show me a LlamaIndex example for <integration/pattern>."
- Coverage checks: "Does an official example already cover this, or do I build from scratch?"

If the user instead wants to *do* the task (write the schema, configure parse, build the index),
that is a product-skill job — see the handoff table at the bottom. This skill points; the product
skills teach.

## Core method: find a recipe for any scenario

Do not rely on a memorized list — the gallery is large and changes. Work from the live catalog hubs
plus search. Follow this sequence:

1. **Classify the scenario into a category** using the map below (e.g. "merged PDF of many invoices"
   → parse + extract cookbooks; "chat over my docs with citations" → index/RAG gallery section).
2. **Search the live catalog** with the doc tools (see "Fetching facts"). Query the *scenario noun*
   plus the *pattern*, e.g. `search_docs "agentic RAG citations"`, `grep_docs "SubQuestionQueryEngine"`.
3. **Confirm against the hub pages.** The two anchor index pages list the curated, currently-valid
   entry points; the gallery sidebar holds the long tail. Fetch the hub `index.md` to get fresh URLs
   rather than pasting remembered ones.
4. **Evaluate fit** (see `references/recipe-finder.md`): does the recipe match the user's data type,
   product (Parse vs Extract vs Index), and pattern — not just a keyword? Surface 1–3 best matches
   with their topic and URL, note any gap, and hand off to the product skill.
5. **If nothing fits**, say so plainly and route to the closest product skill to build from scratch.
   A weak keyword match is worse than an honest "no direct recipe; closest is X."

Keep the answer short: topic + URL + one line on why it fits. Do not paste notebook contents.

### Worked example

Scenario: *"Is there a notebook for pulling line items out of a stack of scanned invoices and then
letting me chat over them?"*

1. **Decompose** into axes (see `references/recipe-finder.md`): data type = scanned PDFs; layer =
   *two legs* — typed extract (line items) + managed index/RAG (chat); pattern = schema-driven
   extraction then retrieval.
2. **Catalog**: mixed → both. Extract leg → Cookbooks (extract); chat leg → Cookbooks (index).
   "Scanned" also touches parse robustness.
3. **Search**: `search_docs "invoice extraction schema"` and `search_docs "index search chat over
   documents"`; `grep_docs "LlamaCloudIndex"` to confirm the managed-index entry point.
4. **Hub-confirm**: fetch `…/llamaparse/cookbooks/index.md` for the current extract + index URLs.
5. **Return, ranked**:
   - Extract leg → extract Starter cookbook (resume_screening pattern, adapt schema to invoice
     fields) — URL — *closest typed-extraction recipe; swap the schema*.
   - Chat leg → index getting_started cookbook — URL — *covers index + search + chat end to end*.
   - **Gap note**: no single notebook chains scanned-invoice extract → chat; combine the two, and
     hand the extract schema design to `llamacloud-extract`, the RAG build to the index product skill.

That gap note is the deliverable's value: an honest "no one recipe does all of it; here are the two
legs" beats forcing a single weak match.

## Coverage checking (the inverse query)

This skill owns *coverage checking* as much as discovery — the question "is this already covered, or
do I build from scratch?" Treat it as the inverse of recipe-finding:

- Run the same decompose + search + hub-confirm method, but the *output* is a verdict, not a recipe.
- **Covered**: a recipe matches all three axes (data type, layer, pattern). Return it and stop —
  route the user to adapt it, not rebuild it.
- **Partially covered**: recipes exist for some legs/axes but not their combination (the worked
  example above). Name what is covered, name the gap, and point each leg at its product skill.
- **Not covered**: no recipe matches the layer/pattern. Say so plainly and route to the nearest
  product skill to build from scratch. Do not invent a match to avoid an empty answer.
- Apply the fit rubric (`references/recipe-finder.md`) before declaring "covered" — a keyword hit on
  a different data type or layer is *not* coverage.

## The two catalogs

**LlamaParse / LlamaCloud cookbooks** — product-centric, organized as *Starter* vs *Advanced*, across
three services: LlamaParse, LlamaExtract, LlamaCloudIndex. Get current URLs from the cookbooks hub
`index.md` (below) — it is the source of truth. (Underlying notebooks historically lived in the
`llama_cloud_services` repo under `examples/{parse,extract,index,batch,sheets,split,misc}/`, but treat
repo paths as rot-prone and confirm via the hub.) Use these for end-to-end LlamaCloud product
scenarios (parse a contract, extract fund data, index + chat).

**LlamaIndex examples gallery** — framework-centric. Curated hub highlights *Agents*,
*Agentic Workflows*, *LLM Integrations*, *Embedding Models*, *Vector Stores*; the full sidebar adds
many more (query/chat engines, retrievers, postprocessors, structured outputs, evaluation,
observability, finetuning, property graph, ingestion, multimodal, managed indexes). Use these for
framework patterns and third-party integrations (which vector store, how to build a ReAct agent,
text-to-SQL, evaluation).

Anchor doc paths (append `index.md` and fetch for fresh URLs):
- `https://developers.llamaindex.ai/llamaparse/cookbooks/`
- `https://developers.llamaindex.ai/python/examples/`

## Scenario → category quick map

A small, durable starting map. It points to the right *category*, then search/hub-fetch from there
for the exact current notebook. The full scenario→category table and fit rubric are in
`references/recipe-finder.md`.

| User scenario | Catalog | Category / anchor |
| --- | --- | --- |
| Parse a PDF/contract to markdown | Cookbooks | parse — Starter (basic), Advanced (multimodal) |
| Tables / charts / scanned / multilingual docs | Cookbooks | parse — table_extraction, get_charts, languages |
| Extract typed fields (resume, invoice, fund report) | Cookbooks | extract — resume_screening, asset_manager_fund_analysis |
| Index + search + chat over my data | Cookbooks | index — getting_started |
| Spreadsheet / messy Excel → rows | Cookbooks | sheets |
| Split one merged file into sections | Cookbooks | split |
| Build a ReAct / function-calling / multi-agent system | Gallery | Agents |
| Workflow-based RAG / agent from scratch / text-to-SQL | Gallery | Agentic Workflows |
| "Which vector store" (Qdrant, Pinecone, pgvector, …) | Gallery | Vector Stores |
| Embedding model choice (Cohere, Voyage, HF, …) | Gallery | Embedding Models |
| Use a specific LLM (Anthropic, Bedrock, Gemini, …) | Gallery | LLM Integrations |
| Evaluation / observability / structured output | Gallery | (search the sidebar long-tail) |

## Fetching facts (live lookups — do not memorize)

URLs, notebook filenames, and gallery sections drift. Always confirm live:

- `mcp__plugin_llamacloud_docs__search_docs` — concept/BM25 search across the docs (best first pass:
  search the scenario noun + pattern).
- `mcp__plugin_llamacloud_docs__read_doc` — read a known doc/cookbook page once you have a path.
- `mcp__plugin_llamacloud_docs__grep_docs` — exact symbol/class/regex search (e.g. a specific engine
  or integration name).
- Fallback: append `index.md` to any `https://developers.llamaindex.ai/...` page URL and WebFetch it
  (e.g. the two anchor hubs above) to get current entry points and URLs.

Treat any notebook filename in this skill as illustrative, not authoritative — verify it still exists.

## Handoffs — this skill points, product skills teach

After surfacing a recipe, hand the actual implementation to the matching product skill:

- Parsing a document → `llamacloud-parse` (cloud) or `liteparse` (local/no-key).
- Typed structured extraction / schema design → `llamacloud-extract`.
- Spreadsheet table extraction → `llamacloud-sheets`.
- Splitting a merged file into sections → `llamacloud-split`.
- Managed index / RAG / retrieval → `llamacloud-index`.
- Account, API key, region, "which product" → core `llamacloud`.

This skill does NOT teach parse/extract/index/split/sheets configuration, schema design, or RAG
construction. It also backs the read-only `cookbook-scout` agent.
