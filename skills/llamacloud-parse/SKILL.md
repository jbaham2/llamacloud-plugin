---
name: llamacloud-parse
description: This skill should be used when the user wants to "parse a PDF/document", "use LlamaParse", choose between "LlamaParse tiers / agentic vs fast vs cost", "configure parse output (markdown/JSON)", or "retrieve/parse results". Owns LlamaParse v2 (GA) parsing decisions and config; defers typed-field extraction, managed retrieval, and local no-key parsing to sibling skills.
version: 0.1.0
---

# LlamaCloud Parse (LlamaParse v2)

LlamaParse is an agentic document parser built for LLM pipelines. It turns messy real-world
files (PDFs, scans, spreadsheets, slides, images) into clean, layout-aware markdown / JSON /
text that downstream models can consume. Use this skill to decide *whether* to parse, *which
tier* to parse with, *how* to configure the job, and *how* to retrieve results.

Parse v2 is the current GA surface. Treat any `/v1/` parse API as legacy / migration-reference
only — never the default for new work.

## First decision: parse, or feed the PDF straight to an LLM?

Multimodal LLMs read a clean, short PDF acceptably on their own. Reach for LlamaParse when the
raw-file approach degrades — and it degrades predictably:

- **Tables / forms / multi-column layouts** — raw rendering loses structure; the LLM
  hallucinates row/column alignment. Parse first.
- **Scans / photos / image-only PDFs** — direct ingestion produces noisy OCR. Parse normalizes
  text, geometry, and reading order.
- **Scale / cost** — feeding raw pages makes token cost scale with *page volume*, not
  information value. Parsing does structural work once, upstream, so the LLM sees only the
  signal.
- **Grounding / citations** — needing page-accurate quotes or bounding boxes is a parse job,
  not an LLM-prompt job.

Skip parsing only for a handful of short, simple, text-native documents where speed beats
fidelity. Otherwise parse. See `references/tier-selection.md` for the complexity-first decision.

## Second decision: which tier?

Choose by **document complexity, not price**. The live tiers (lowest → highest capability):
**Fast**, **Cost Effective**, **Agentic**, **Agentic Plus** — plus the **Cost Optimizer** for
mixed documents. Quick orientation:

- Plain text, high volume, no markdown/tables needed → **Fast**.
- Mostly prose with simple tables → **Cost Effective**.
- Real-world PDFs (tables, scans, multi-column, charts) → **Agentic** (the recommended default).
- Dense financial/scientific docs, equations, mission-critical accuracy → **Agentic Plus**.
- One document whose pages vary wildly in complexity → enable **Cost Optimizer** instead of
  picking a single tier; it routes each page to the right tier in parallel.

Tier choice has capability consequences (e.g. markdown and chart parsing are not available on
the lowest tier). The full decision table, tradeoffs, and these constraints live in
`references/tier-selection.md`.

## Third decision: configuration surface

Configure across four families — pick deliberately, do not over-configure:

- **Input controls** — point at a file or URL; optionally restrict pages, cap page count, or
  geometrically crop repeating chrome. File-type hints exist for HTML, spreadsheets, slides,
  and camera-photo images.
- **Output format** — markdown (default, the right choice for most LLM pipelines), JSON with
  typed items, plain/spatial text, per-table spreadsheets, image assets, or bounding boxes.
- **Processing options** — Cost Optimizer, chart parsing, natural-language custom prompts,
  content-skipping (watermarks/hidden text), OCR languages, table tuning.
- **Result retrieval shape** — what fields come back (see below).

Config *judgment* and the gotchas that bite (cache-busting, crop-box-is-geometric, fast-tier
markdown limits, formula recompute cost) are in `references/configuration.md`.

## Fourth decision: retrieving results

Results return in two shapes: **content fields** deliver parsed data inline (text, markdown,
typed JSON items, metadata); **metadata fields** return presigned download URLs for large
payloads you want to stream or defer. Key judgment:

- **Request only what you need.** Most LLM pipelines retrieve markdown alone. Over-expanding
  bloats responses.
- **Re-retrieve without re-parsing.** A parsed job can be fetched again later with different
  fields requested — no second parse, no second charge.
- **Check size before downloading** large files via the metadata/URL path.
- **Presigned URLs expire** — download promptly or request fresh ones.

Retrieval gotchas (fast-tier markdown errors, grounded-items sidecars) are in
`references/configuration.md`.

## Minimal, illustrative snippets

These show shape only. Confirm the exact current package name, client class, and parameter
names live (see "Fetching facts") rather than copying these verbatim.

Python (package name has shifted between `llama-cloud` and `llama-cloud-services` — verify live):

```python
from llama_cloud_services import LlamaParse  # confirm current package/class via MCP

parser = LlamaParse(api_key="...", tier="agentic", result_type="markdown")
result = parser.parse("report.pdf")
```

TypeScript:

```ts
import { LlamaParse } from "llama-cloud-services"; // confirm current package name live

const parser = new LlamaParse({ apiKey: "...", tier: "agentic", resultType: "markdown" });
const result = await parser.parse("report.pdf");
```

For exact, current signatures (method names, parameter keys, tier identifiers, expand values),
use the LlamaCloud MCP — do not trust these illustrative snippets for parameter names.

## Fetching facts

Signatures, package names, tier identifiers, parameter keys, and limits drift. Look them up live:

- `mcp__plugin_llamacloud_docs__search_docs` — concept / BM25 search.
- `mcp__plugin_llamacloud_docs__read_doc` — read a known doc page.
- `mcp__plugin_llamacloud_docs__grep_docs` — exact symbol / regex search.
- Fallback: append `index.md` to any `https://developers.llamaindex.ai/...` page URL and WebFetch
  it.

Anchor docs for this pillar:

- `https://developers.llamaindex.ai/llamaparse/parse/`
- `https://developers.llamaindex.ai/llamaparse/parse/guides/tiers/`
- `https://developers.llamaindex.ai/llamaparse/parse/guides/configuring-parse/`
- `https://developers.llamaindex.ai/llamaparse/parse/guides/retrieving-results/`

## Does NOT cover (hand off)

- **Structured, typed field extraction** (schemas, pulling specific values out) → `llamacloud-extract`.
- **Managed RAG / retrieval / indexing** over parsed content → `llamacloud-index`.
- **Local / offline parsing with no API key** → `liteparse`.
- **Account, API key, and region setup** → core `llamacloud`.
