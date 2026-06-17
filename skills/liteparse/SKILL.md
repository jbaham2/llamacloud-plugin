---
name: liteparse
description: This skill should be used when the user wants to "use LiteParse", "parse locally without an API key", do "offline/OSS document parsing", or work with the "LiteParse library/CLI/server/WASM". Owns local/OSS no-key document parsing and the LiteParse-vs-cloud-LlamaParse decision; defers cloud/agentic/scaled parsing, typed extraction, and managed RAG to sibling skills.
version: 0.1.0
---

# LiteParse: local, OSS, no-API-key document parsing

LiteParse is an open-source, Rust-based document parser that extracts text with
spatial layout, per-line bounding boxes, and page screenshots — entirely on the
local machine. No cloud calls, no LLMs, no API key. Reach for it when privacy,
air-gapping, latency, cost, or tight dev loops matter more than the highest
achievable parse quality on messy documents.

This is distinct from cloud **LlamaParse** (the hosted Parse v2 service, which
needs a `LLAMA_CLOUD_API_KEY` and uses models for hard layouts). LiteParse trades
that ceiling for zero dependencies and full local control.

## When to use LiteParse vs cloud LlamaParse

Choose LiteParse when:
- **Privacy / air-gapped**: documents cannot leave the machine or network.
- **Dev loops**: fast, free, deterministic local iteration while building a pipeline.
- **Cost / scale-of-volume**: parsing high volumes where per-page cloud credits add up.
- **Simple-to-moderate docs**: digital PDFs, Office files, clean scans where layout
  extraction + OCR is enough.
- **Browser / edge**: parsing must run client-side (WASM) with no server at all.

Hand off to **cloud LlamaParse** (sibling skill `llamacloud-parse`) when the job
needs the highest-quality, model-driven, or agentic parsing — dense tables,
complex multi-column layouts, handwriting, or chart/figure understanding at scale.
The two compose well: prototype locally with LiteParse, escalate hard documents to
the cloud. See `references/liteparse-vs-llamaparse.md` for the full decision table.

## Choosing a usage mode

LiteParse ships the same engine across four modes — pick by where the code runs:

- **Library** — embed in a Python, TypeScript/Node, or Rust app. Default choice for
  pipelines and services. Construct a `LiteParse` instance, call its parse method,
  read text / pages / bounding boxes.
- **CLI** — one-off or scripted parsing from the shell; good for batch jobs and
  quick inspection without writing code.
- **Server** — run LiteParse as a long-lived local HTTP service so multiple
  clients/languages share one parser process (avoids re-init cost per call).
- **WASM** — run fully in the browser (`@llamaindex/liteparse-wasm`) for client-side,
  zero-server parsing where documents never touch a backend.

Mode-selection trade-offs (init cost, concurrency, deployment) live in
`references/liteparse-vs-llamaparse.md`.

## Minimal usage

Confirm the exact current package name and signatures via the MCP before writing
production code — package names and parameters shift. Illustrative only:

Python (`liteparse`):
```python
from liteparse import LiteParse

parser = LiteParse()                    # construct once, reuse
result = parser.parse("document.pdf")   # also accepts raw bytes
print(result.text)                      # layout-preserving text
for page in result.pages:               # per-page items with bounding boxes
    print(page.page_num, len(page.text_items))
```

TypeScript (`@llamaindex/liteparse`):
```typescript
import { LiteParse } from "@llamaindex/liteparse";

const parser = new LiteParse({ ocrEnabled: true });
const result = await parser.parse("document.pdf");
console.log(result.text);
```

Common knobs (names vary by binding — verify live): OCR enable/language/server URL,
render `dpi`, `output_format` (plain text vs JSON with `x/y/width/height`),
`target_pages` (e.g. `"1-10"`), and `password` for encrypted PDFs. Built-in
Tesseract OCR handles scans; a custom OCR server can be pointed to instead.

## Outputs

- **Layout-preserving text** — reading-order text that reflects page geometry.
- **Bounding boxes** — per-line coordinates for every text item (citation,
  highlighting, downstream layout logic).
- **Page screenshots** — PNG page renders, useful as image inputs to multimodal LLMs.

For chunking, embedding, and retrieval over these outputs, this skill stops at
parse; the downstream RAG pipeline belongs to sibling skills.

## Fetching facts (do not trust memory for these)

Package names, class/method signatures, parameter names, CLI flags, and supported
formats change. Look them up live:

- `mcp__plugin_llamacloud_docs__search_docs` — concept/BM25 search.
- `mcp__plugin_llamacloud_docs__read_doc` — read a known doc page.
- `mcp__plugin_llamacloud_docs__grep_docs` — exact symbol/regex search.
- Fallback: append `index.md` to any `https://developers.llamaindex.ai/...` URL and
  WebFetch it.

Anchor docs for this pillar:
- `https://developers.llamaindex.ai/liteparse/` — overview, modes, formats.
- `https://developers.llamaindex.ai/liteparse/guides/library-usage/` — library API.
- Browse `https://developers.llamaindex.ai/liteparse/guides/` for CLI, server, WASM,
  and OCR specifics (verify exact subpaths via the MCP rather than guessing).

## Boundaries (defer to siblings)

- Highest-quality / agentic / scaled **cloud** parsing → `llamacloud-parse`.
- Typed/schema-driven **extraction** from documents → `llamacloud-extract`.
- Managed **RAG / indexing** over parsed content → `llamacloud-index`.

Keep LiteParse scoped to local, no-key parsing and the local-vs-cloud decision.
