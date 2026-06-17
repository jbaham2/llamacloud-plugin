---
name: llamacloud
description: This skill should be used when the user is getting started with LlamaCloud or LlamaParse, asks "which LlamaCloud product do I need", "how do I set up LlamaCloud", "get a LlamaCloud / LlamaParse API key", asks about LlamaCloud regions / EU data residency / pricing / credits, or needs routing across the LlamaCloud product family (Parse, Extract, Classify, Split, Sheets, Cloud Index). Acts as the router and onboarding entry point; defers product-specific depth to the dedicated parse/extract/index/classify/split/sheets skills.
version: 0.1.0
---

# LlamaCloud — Router & Onboarding

LlamaCloud is LlamaIndex's document-AI platform. This skill is the **entry point**: it picks the
right product for a task, gets the user authenticated and configured, and hands off to the
specialized skill. It holds the *cross-product mental model and onboarding judgment* — it does
**not** teach any single product in depth.

## When to route elsewhere

After identifying the product, defer to the owning skill (it auto-triggers on product verbs):

| User intent | Product | Skill to defer to |
|---|---|---|
| Turn PDFs/scans into clean LLM-ready text/markdown | **Parse** | `llamacloud-parse` |
| Pull structured JSON matching a schema out of documents | **Extract** | `llamacloud-extract` |
| Build a hosted/managed vector pipeline for RAG | **Cloud Index** | `llamacloud-index` |
| Route documents into categories by rule | **Classify** | `llamacloud-classify` |
| Segment one combined/concatenated PDF into logical sub-documents | **Split** | `llamacloud-split` |
| Extract tables/rows from messy spreadsheets | **Sheets** | `llamacloud-sheets` |
| Parse locally/offline, no API key, OSS | **LiteParse** | `liteparse` |
| Self-host the platform (BYOC/Helm/K8s) | — | `llamacloud-self-hosting` |
| Build RAG/agents in OSS code (not managed) | — | `llamaindex-framework`, `llamaindex-workflows` |

If the task spans products, stay here long enough to sequence them, then hand off in order.

## The product mental model

Parse is the **foundation most pipelines build on**: document → clean text/markdown. Its output
then feeds:
- **Extract** (structured fields), **Classify** (category routing), **Split** (segmentation), and
- **Cloud Index** (chunk → embed → retrieve for RAG).

Decision shortcuts:
- Need *typed fields* (invoice total, parties, dates) → **Extract**, not Parse. Extract parses
  internally; do not chain Parse→Extract manually unless you need the raw text too.
- Need *semantic search / RAG over a corpus* → **Cloud Index** (managed) or the OSS
  `llamaindex-framework` (self-managed). Choose managed when you want hosted ingestion + retrieval
  without running a vector DB; choose the framework when you need full control or local infra.
- Just need *clean text/markdown* for your own downstream LLM call → **Parse**.
- A pile of mixed document types arriving together → **Classify** (route) and/or **Split**
  (segment) before Parse/Extract.

See `references/product-routing.md` for the full decision tree, multi-product pipelines, and the
managed-vs-OSS and Parse-vs-Extract boundary calls.

## Version posture (authoritative — apply everywhere)

- **Parse v2** and **Extract v2** are the **primary GA surface**. Generate v2 code by default.
- Their `/v1/` doc trees are **legacy / migration reference only** — never the default path.
- **Cloud Index v2** is **beta and deferred** — teach the stable Cloud Index surface; flag v2 as beta.
- **LiteParse** is local/OSS with **no API key**.
- Classify, Split, and Sheets are **beta** — set expectations accordingly.

Never present a legacy (`/v1/`) or beta path as the default. This posture is the plugin's
default-surface directive (durable); the **GA/beta labels themselves drift** — when a product's
status matters to the answer, confirm the current GA/beta state live via the docs MCP before relying
on it.

## Onboarding sequence

Run this order when a user is setting up (the `/llamacloud-setup` command wraps it):

1. **Account + key.** Sign in at the LlamaCloud console and create an API key. The whole platform
   uses one credential, exposed to code as `${LLAMA_CLOUD_API_KEY}`. Never hardcode it; never echo it.
2. **Pick a region** — this is a **one-time, non-reversible** choice:
   - **NA** (`api.cloud.llamaindex.ai`) — gets new features first; default.
   - **EU** (`api.cloud.eu.llamaindex.ai`) — data stored & processed in-region; pick for GDPR /
     EU data-residency. Usable from anywhere if residency demands it. **Cannot migrate later.**
   The SDK targets NA by default; point it at the EU base URL when EU is chosen.
3. **Install the SDK.** One package serves all products. Confirm the exact current package name and
   version live (it has shifted — see "Fetching facts" below) before pinning it.
4. **First call.** Verify the key with the smallest possible operation in the chosen product, then
   hand off to that product's skill.

See `references/onboarding.md` for the step-by-step setup checklist, the env-var/region wiring
pattern (Python + TypeScript), key security/rotation guidance, and the pricing/credits mental model.

## Fetching facts (do not hardcode)

Treat these as **live lookups**, never memorized: exact package names/versions, SDK class and method
signatures, base-URL constants, pricing/credit numbers, and rate limits. They drift. Fetch them via:

- **Docs MCP** (wired, no auth): `mcp__plugin_llamacloud_docs__search_docs` for concepts,
  `mcp__plugin_llamacloud_docs__read_doc` for a known page, `mcp__plugin_llamacloud_docs__grep_docs`
  for an exact symbol.
- **Clean-markdown fallback:** append `index.md` to any `https://developers.llamaindex.ai/...` page
  URL and `WebFetch` it.

Anchor pages: `/llamaparse/` (overview), `/llamaparse/general/api_key/`,
`/llamaparse/general/regions/`, `/llamaparse/general/pricing/`.

## SDK packages (verify live before pinning)

Two layers exist and the naming has changed across releases, so **confirm before quoting**:
- A high-level services SDK (commonly `llama-cloud-services` in Python) exposing `LlamaParse`,
  `LlamaExtract`, etc. — the recommended surface for v2.
- A lower-level generated client (`llama-cloud` / `@llamaindex/llama-cloud`).

Default to the high-level services SDK for Parse/Extract v2; fetch the exact current install command
from the docs rather than asserting it from memory.

## Additional resources

- **`references/product-routing.md`** — full routing decision tree, multi-product pipelines, the
  managed-vs-OSS and Parse-vs-Extract-vs-Index boundaries, and anti-patterns.
- **`references/onboarding.md`** — setup checklist, region/env wiring, key security, pricing/credits
  mental model, common first-run failures.
