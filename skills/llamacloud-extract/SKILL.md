---
name: llamacloud-extract
description: This skill should be used when the user wants to "extract structured data from a document", "design/validate an extraction schema", write a "Pydantic/JSON/Zod extract schema", configure "LlamaExtract options/citations", or "pull typed fields from PDFs". Owns LlamaExtract v2 schema design and configuration judgment; defers raw text/markdown parsing, RAG/retrieval, and account/key/region setup to sibling skills.
version: 0.1.0
---

# LlamaCloud Extract (v2)

LlamaExtract turns unstructured documents (PDFs, images, text) into typed output
that conforms to a schema you define. Use it when the goal is **schema-conformant
data** for a database, dashboard, spreadsheet, or downstream model — not document
text for a human or a RAG pipeline.

## Is Extract the right tool?

- **Yes** — you know the fields you want and need them typed and validated:
  invoice fields, contract terms, resume attributes, table rows as records.
- **No, you want raw text/markdown** of the document → use `llamacloud-parse`.
- **No, you want to search/answer questions over many documents** (retrieval,
  RAG) → use `llamacloud-index`.
- **Account, API key, or region setup** is not covered here → core `llamacloud`.

Extract is built on parsing, but its contract is *schema conformance and data
quality*, which is why schema design is where most of the work and most of the
wins are.

## Version posture

Extract **v2 is the primary, GA path** — design all new work against it. The
`/v1/` extract API is legacy and exists only as a migration reference; never make
it the default. If you encounter v1 code, treat porting it to v2 as the goal.

## Workflow

1. **Confirm Extract fits** (above). If the user really wants raw text or
   retrieval, hand off to the sibling skill.
2. **Design the schema first.** This dominates output quality. Apply the durable
   rules in `references/schema-design.md`: object root, shallow nesting, strong
   field names, descriptions that carry format/examples, optional booleans and
   integers, prune auto-generated fields, start minimal and iterate.
3. **Choose how the schema is applied.** Pick the extraction target by document
   shape (per-document / per-page / per-row), a tier by complexity and stakes,
   and decide on system prompt, page range, and citations. See
   `references/options-and-tiers.md`.
4. **Run, inspect, refine.** Turn citations on while tuning; fix misses by
   sharpening field descriptions before touching the tier.
5. **Validate the result** against your typed model in code and post-process
   anything the schema can't enforce (defaults, formats).

## Schema design (the core skill)

Author the schema as a typed model in your language — **Pydantic** in Python,
**Zod** in TypeScript — or hand-write JSON Schema; all compile to JSON Schema for
the API. Key durable rules:

- Root is an **object**; wrap repeating entities in a field (arrays can't be the
  root).
- Keep nesting to ~3–4 levels; flatten rather than mirror the page layout.
- Names become JSON keys — choose stable, query-friendly ones.
- **Descriptions guide the model** — encode format and examples there.
- JSON Schema `default`/`format` are **not enforced** — encode them in
  descriptions or post-process.
- Make **booleans and integers optional** so "absent" isn't confused with
  `false`/`0`.

Full rules, the optional-field syntaxes, caps, and the split-and-merge strategy
for oversized schemas are in `references/schema-design.md`.

## Configuration (target, tier, options)

- **Target** by document shape: one record per document, per page, or per table
  row. A mismatched target is a top failure mode for repeating entities.
- **Tier** by complexity/stakes: the agentic default fits most documents; drop to
  cost-effective only after verifying accuracy; reserve the top tier for dense,
  high-stakes layouts.
- **System prompt** for *global* rules (disambiguation, conventions); keep
  field-level guidance in descriptions.
- **Page range** to scope large documents.
- **Citations / reasoning / confidence** for auditability and for diagnosing
  wrong extractions — the best debugging tool — at some latency cost.

Decision tables and the tuning loop are in `references/options-and-tiers.md`.

## Minimal examples (illustrative — verify signatures live)

Python (confirm the current package name via the MCP — it has shifted between releases):

```python
from pydantic import BaseModel
from typing import Optional

class Invoice(BaseModel):
    invoice_number: str           # field name = JSON key
    total_amount: float
    paid: Optional[bool] = None   # optional so absent != False

# Submit `Invoice` as the schema to an extraction agent, then validate the
# returned dict back through Invoice(**result) and post-process.
```

TypeScript (Zod):

```ts
import { z } from "zod";

const Invoice = z.object({
  invoice_number: z.string(),
  total_amount: z.number(),
  paid: z.boolean().nullable(),   // optional/absent-safe
});
```

These show *shape*, not the current API surface. Confirm the exact package name,
agent/client classes, and method/parameter names via MCP (below) — the SDK
package name has shifted between releases, so do not assert it from memory.

## Fetching facts

Look up exact, stale-prone details live (package name, class/method signatures,
parameter and enum values, the current caps, page-range syntax, pricing):

- `mcp__plugin_llamacloud_docs__search_docs` — concept/BM25 search.
- `mcp__plugin_llamacloud_docs__read_doc` — read a known doc page.
- `mcp__plugin_llamacloud_docs__grep_docs` — exact symbol/regex search.
- Fallback: append `index.md` to any `https://developers.llamaindex.ai/...` page
  URL and WebFetch it.

Anchor docs for this pillar:

- `https://developers.llamaindex.ai/llamaparse/extract/`
- `https://developers.llamaindex.ai/llamaparse/extract/guides/schema_design/`
- `https://developers.llamaindex.ai/llamaparse/extract/guides/options/`

For exact, current SDK signatures, prefer the MCP over any snippet in this skill.

## Boundaries

- Raw text / markdown extraction → `llamacloud-parse`.
- Retrieval / RAG over documents → `llamacloud-index`.
- API keys, regions, account setup → core `llamacloud`.
