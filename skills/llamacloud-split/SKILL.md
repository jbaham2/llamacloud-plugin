---
name: llamacloud-split
description: This skill should be used when the user wants to "split a document", "segment a concatenated/merged PDF", "break one file into logical sub-documents", or do "page/section splitting" of a single combined file into typed sections. It owns LlamaCloud Split (page-range segmentation of one bundled file) and defers deciding each section's type to llamacloud-classify and typed field extraction to llamacloud-extract.
version: 0.1.0
---

# LlamaCloud Split

Segment a single concatenated/bundled file (many documents merged into one PDF) into
logical sub-documents. Split scans pages, assigns each to a user-defined category, and
groups consecutive same-category pages into contiguous segments with page ranges and a
confidence rating. Use it as the *demux* step before per-document Classify and Extract.

## When this skill applies

Reach for Split when the input is **one file that contains several documents** and the goal
is to find the **cut points** — where one sub-document ends and the next begins. Typical
phrasings: "split this combined PDF," "segment a merged loan package," "break this bundle
into separate contracts/reports," "page splitting by section."

Do NOT use Split when:
- The file is already a single document and the question is "what type is it?" → that is
  labeling, not segmentation. Hand off to `llamacloud-classify`.
- The goal is pulling structured fields (amounts, dates, parties) → `llamacloud-extract`.
- The goal is raw LLM-ready text/markdown from the file → `llamacloud-parse`.
- API keys, project setup, or "which LlamaCloud product" → core `llamacloud`.

Split *uses* classification internally to decide boundaries, which makes it easy to confuse
with Classify. The crisp line: **Split decides WHERE to cut** one combined file into page
ranges; **Classify decides the TYPE/label** of an already-separated document. See
`references/split-strategy.md` for the full boundary and sequencing.

## Version posture (read first)

Split is **beta** and subject to breaking changes. Two consequences to set expectations on:

1. **REST-only today.** As of this writing there is no first-class Split client in the
   `llama-cloud-services` SDK — all operations go through the LlamaCloud REST API directly
   (define categories → create a split job → poll → retrieve segments). Confirm live before
   asserting an SDK path exists; a Split client may be added later.
2. **Shapes can move.** Endpoint paths, request/response field names, and the job lifecycle
   are not a stable contract yet. Teach the *flow*, look up the *exact shapes* live.

## The Split flow (conceptual)

1. **Upload the file** to LlamaCloud via the Files API (same file handle used across Parse /
   Extract / Classify). Get back a file id.
2. **Define categories** — a list of `{name, description}` pairs in natural language (e.g.
   `invoice`: "a vendor bill with line items and a total due"). Descriptions are the levers
   that steer page assignment; invest in them. Optionally allow an "uncategorized" bucket.
3. **Create a split job** referencing the file id and the categories.
4. **Poll the job** until it reaches a terminal state (success / error). Splitting is async.
5. **Retrieve segments** — each segment carries the assigned category, the **page range**
   (start/end), and a **confidence** rating (high / medium / low). Use these ranges to slice
   the original file into per-document inputs for the next stage.

The durable contract is this five-step sequence. The literal endpoint paths and JSON field
names are stale-prone — fetch them live (see "Fetching facts").

## Minimal REST snippets (illustrative)

Both samples show the same create → poll → read shape. They omit exact paths/fields on
purpose — resolve those live. Treat them as structure, not signatures.

Python (`httpx` or `requests`):

```python
import os, time, httpx

base = "https://api.cloud.llamaindex.ai"  # confirm host/path live
h = {"Authorization": f"Bearer {os.environ['LLAMA_CLOUD_API_KEY']}"}

# 1) upload file -> file_id (Files API); 2) define categories
categories = [
    {"name": "invoice",  "description": "a vendor bill with line items and a total due"},
    {"name": "contract", "description": "a signed agreement with parties and clauses"},
]

# 3) create split job (path + body shape: look up live)
job = httpx.post(f"{base}/.../split/jobs",
                 headers=h, json={"file_id": file_id, "categories": categories}).json()

# 4) poll until terminal
while job["status"] not in ("SUCCESS", "ERROR"):
    time.sleep(2)
    job = httpx.get(f"{base}/.../split/jobs/{job['id']}", headers=h).json()

# 5) read segments: each has category, page range, confidence
for seg in job["segments"]:
    print(seg["category"], seg["page_start"], seg["page_end"], seg["confidence"])
```

TypeScript (`fetch`):

```ts
const base = "https://api.cloud.llamaindex.ai"; // confirm host/path live
const h = { Authorization: `Bearer ${process.env.LLAMA_CLOUD_API_KEY}` };
const categories = [
  { name: "invoice",  description: "a vendor bill with line items and a total due" },
  { name: "contract", description: "a signed agreement with parties and clauses" },
];

let job = await (await fetch(`${base}/.../split/jobs`, {
  method: "POST",
  headers: { ...h, "Content-Type": "application/json" },
  body: JSON.stringify({ file_id, categories }),
})).json();

while (!["SUCCESS", "ERROR"].includes(job.status)) {
  await new Promise(r => setTimeout(r, 2000));
  job = await (await fetch(`${base}/.../split/jobs/${job.id}`, { headers: h })).json();
}
for (const s of job.segments) console.log(s.category, s.page_start, s.page_end, s.confidence);
```

## Sequencing with Classify and Extract

Split is the **first** demux step in a multi-document pipeline:

```
one bundled file
  → Parse (LLM-ready text, if needed)        # llamacloud-parse
  → SPLIT (page ranges per sub-document)      # THIS skill
  → Classify (confirm/refine each doc's type) # llamacloud-classify
  → Extract (typed fields per doc)            # llamacloud-extract
```

Common shortcut: Split already assigns a category per segment, so for simple bundles you can
go straight Split → Extract and skip a separate Classify pass. Add Classify when Split's
categories are coarse, when you need a different/finer taxonomy for routing, or when
low-confidence segments need a second opinion. Decision table and failure modes:
`references/split-strategy.md`.

## Fetching facts (do not trust memory)

Exact endpoint paths, request/response field names, job status enum, category schema, page
limits, and beta credit costs change — look them up live every time:

- `mcp__plugin_llamacloud_docs__search_docs` — concept / BM25 search ("split job", "segment").
- `mcp__plugin_llamacloud_docs__read_doc` — read a known page (the Split overview).
- `mcp__plugin_llamacloud_docs__grep_docs` — exact symbol / regex (endpoint path, field name).
- Fallback: append `index.md` to any `https://developers.llamaindex.ai/...` URL and WebFetch
  it. Anchor: `https://developers.llamaindex.ai/llamaparse/split/index.md`.

For whether `llama-cloud-services` has gained a Split client, confirm the current package and
its API live rather than asserting it — the package name has shifted historically.

## Reference

- `references/split-strategy.md` — when splitting helps vs. hurts, the Split-vs-Classify
  boundary, sequencing with Classify/Extract, confidence handling, and failure modes.
