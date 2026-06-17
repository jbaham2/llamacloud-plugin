---
name: llamacloud-classify
description: This skill should be used when the user wants to "classify documents", "route documents by type", define "document classification rules", or "categorize incoming files" with LlamaCloud Classify. Owns natural-language classification and intake routing; defers segmenting one file into sub-docs to llamacloud-split, typed field extraction to llamacloud-extract, and raw text to llamacloud-parse.
version: 0.1.0
---

# LlamaCloud Classify

Assign one user-defined category to each document using natural-language rules, with a
confidence score and a reasoning explanation per result. Use Classify as a router: send a
mixed inbox of invoices, receipts, contracts, and statements to the correct per-type
downstream pipeline.

## When to reach for Classify

Reach for Classify when the goal is to **label or route whole documents** by type before
doing anything expensive or type-specific with them:

- Intake routing — direct each incoming file to the pipeline that matches its kind.
- Pre-processing gate — pick the right extraction schema or parse preset per document type.
- Dataset curation — pull a labeled subset (e.g. "only 10-Ks") out of a large pile.

Classify answers "**what kind of document is this?**" It does not pull fields out, does not
turn pages into text, and does not break a bundle into pieces. Those are siblings (see
Boundaries).

The value is **avoiding work**: classification is cheap relative to extraction or parsing,
so a Classify gate at the front of a pipeline lets the expensive, type-specific stages run
only on the documents they apply to — and lets each stage assume it is seeing the type it
was built for. A single mixed inbox becomes a set of clean, single-type streams.

## Beta posture (read before designing)

Classify is **beta** and subject to breaking changes. Design for that:

- **Never hard-wire Classify as a hard SLA dependency without a fallback.** Treat the
  category as a hint, not a contract.
- **Gate routing on the confidence score.** Route high-confidence results automatically;
  send low-confidence (and any "none"/catch-all) results to a human-review or default
  queue. Pick the threshold empirically on a sample, do not guess.
- Pin a known-good SDK version in production and re-test when upgrading, since signatures
  may shift during beta.

See `references/classification-design.md` for the confidence-gating pattern in full.

## Core workflow

1. **Upload** the files to LlamaCloud (or pass file IDs you already have).
2. **Define rules** — a list of categories, each with a `type` label and a natural-language
   `description` of what content belongs to it.
3. **Choose a mode** — text-only vs. vision-based (see Mode selection).
4. **Run the classify job** and read back, per file: predicted `type`, `confidence`
   (0.0–1.0), and `reasoning`.

A rule's `type` label is constrained to alphanumerics, spaces, hyphens, and underscores
(e.g. `invoice`, `10-K`, `purchase-order`). The `description` is free natural language and
is where almost all of the accuracy comes from — invest there.

Each result carries three fields, all of which matter operationally:

- `type` — the predicted category. This is what downstream branches on.
- `confidence` — 0.0–1.0. This is what you gate routing on (see Beta posture). Never act on
  `type` while ignoring `confidence`.
- `reasoning` — the model's stated justification. This is the primary tool for debugging
  misclassifications: it tells you *why* a document landed where it did, so you know which
  rule description to sharpen.

Plan the downstream branch for each `type` up front, including what happens on the
catch-all category and on low confidence, before wiring routing — those two lanes are where
unhandled documents leak.

## Mode selection

Two modes trade cost against visual understanding. Choose by **what carries the signal**,
not by file format:

| Signal lives in…                                  | Mode        |
| ------------------------------------------------- | ----------- |
| Plain text, headers, field labels, layout-in-text | Fast        |
| Handwriting, scans, stamps, charts, logos, photos | Multimodal  |

Fast is cheaper; Multimodal costs more per page but reads visual cues. If text alone
reliably distinguishes your categories, stay on Fast. Move to Multimodal only when the
distinguishing feature is something a text extractor would lose. Confirm current
per-page credit costs live (see Fetching facts) — do not assume fixed numbers.

## Writing good rules

Rules are prompts. The model picks the category whose description best matches the
document, so descriptions must be **discriminative**, not merely accurate.

- **Name the distinguishing features**, especially the fields/structure unique to that type
  ("contains an invoice number, bill-to block, and line-item totals").
- **Contrast against look-alikes** when two types are close ("a receipt is a short POS
  purchase record — unlike an invoice, it has no bill-to section or payment terms").
- **Add a catch-all category** (e.g. `other`) so genuinely-unmatched documents land
  somewhere explicit instead of being forced into the nearest wrong bucket.
- **Iterate on a labeled sample.** Start simple, run, inspect the `reasoning` on
  misclassifications, then sharpen the description that lost. The reasoning field is the
  primary debugging tool.

Common rule-writing mistakes: writing descriptions that are accurate but not
*discriminative* (every type "contains text and numbers"); omitting the catch-all so
off-distribution documents are forced into a wrong bucket; and over-splitting into
categories whose only difference downstream is the label. Resist adding a category until a
downstream step actually treats it differently.

Taxonomy design (mutually-exclusive + collectively-exhaustive categories, handling
multi-type documents, granularity, and the full failure-mode table) is covered in
`references/classification-design.md`. Consult it before designing a category set or
diagnosing low accuracy.

## Minimal shape

Python (`llama-cloud-services`) — illustrative; confirm the exact current package, client,
and method names via the MCP before relying on them:

```python
# from the llama-cloud-services classify client
rules = [
    {"type": "invoice",  "description": "Has an invoice number, bill-to block, line items, and payment terms."},
    {"type": "receipt",  "description": "Short POS purchase record; no bill-to section or payment terms."},
    {"type": "other",    "description": "Anything not matching the categories above."},
]
result = client.classifier.classify(files=[...], rules=rules, mode="FAST")
for item in result.items:
    print(item.result.type, item.result.confidence, item.result.reasoning)
```

TypeScript follows the same shape (define `rules`, submit with a `mode`, read
`type` / `confidence` / `reasoning` per item). For exact signatures, parameter names, and
the current package, look them up live rather than copying the above.

## Worked routing pattern

A concrete intake router for a mixed accounts-payable inbox, to make the pieces concrete:

1. **Define the taxonomy** around what downstream does, not around document trivia. Suppose
   three pipelines exist: one for vendor invoices, one for expense receipts, one for
   everything else (manual review). That yields three categories — `invoice`, `receipt`,
   `other` — no finer, because nothing downstream branches finer.
2. **Write discriminative descriptions.** `invoice`: "has an invoice number, a bill-to
   block, line items, and payment terms." `receipt`: "short POS purchase record; *unlike an
   invoice* it has no bill-to section or payment terms." `other`: "anything not matching the
   above." The contrast clause on `receipt` is what prevents the two from colliding.
3. **Pick a mode.** If the inbox is born-digital PDFs with selectable text, Fast suffices.
   If it includes phone-photographed paper receipts, route those to Multimodal — or run a
   cheap pre-sort and send only the scanned subset to Multimodal.
4. **Calibrate confidence on a labeled sample** before going live: find the threshold where
   auto-routing hits the acceptable error rate.
5. **Route in three lanes.** High-confidence `invoice`/`receipt` → their pipelines.
   Low-confidence or `other` → human-review queue. Job error → retry, then dead-letter.
6. **Log `type`, `confidence`, `reasoning`** for every file. When accuracy drifts, read the
   reasoning on the misses and sharpen the one description that lost — do not rewrite the
   whole rule set.

If step 1's inbox were instead **one merged PDF** containing many stapled-together
documents, the router would be wrong from the start: Split it first, then run this flow on
each piece. See Boundaries.

## Boundaries (defer to siblings)

- **One file holds several concatenated documents** (a scanned batch, a merged PDF) →
  segment it **first** with `llamacloud-split`, then run Classify on each piece. Classify
  assigns a single type to a whole file; it will not carve a bundle apart.
- **Need typed fields out of the document** (totals, dates, parties) → `llamacloud-extract`.
  A common pattern is Classify → pick schema by type → Extract.
- **Need raw text / markdown of the document** → `llamacloud-parse`.
- **Account setup, API keys, choosing among LlamaCloud products** → core `llamacloud`.

## Fetching facts

Treat package names, client/method signatures, parameter names, mode enum values, and
per-page credit costs as live lookups, not memorized constants:

- `mcp__plugin_llamacloud_docs__search_docs` — concept/BM25 search.
- `mcp__plugin_llamacloud_docs__read_doc` — read a known doc page.
- `mcp__plugin_llamacloud_docs__grep_docs` — exact symbol/regex search.
- Fallback: append `index.md` to any `https://developers.llamaindex.ai/...` page URL and
  WebFetch it.

Anchor docs for this pillar:

- Overview: `https://developers.llamaindex.ai/llamaparse/classify/`
- SDK guide: `https://developers.llamaindex.ai/llamaparse/classify/sdk/`
- API reference: `https://developers.llamaindex.ai/reference/resources/classify/`
