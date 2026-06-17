# Extraction options & tiers — decision guidance

Set the schema first (see schema-design.md); these options tune *how* the schema
is applied. Exact enum/parameter values shift between releases — confirm names
live via MCP. The judgment below is durable.

## Extraction target — pick by document shape

The target controls how the schema maps onto the document. This is the single
most impactful option after the schema itself.

| Document shape | Target | Why |
|---|---|---|
| One coherent document → one record (an invoice, a contract, a resume) | per-document | Schema applied once over the whole doc; returns a single object. |
| Same fields repeat on each page (per-page forms, statements) | per-page | Schema applied independently per page; returns one record per page. |
| A table/ordered list where each row is its own entity (line items, a roster) | per-row | Schema describes one row's fields; returns one record per row. Pair with a schema that models a *single* entity, not the whole table. |

Mismatched target is a common failure: modeling a list of entities with a
per-document target forces everything into one object and the model drops or
merges rows. Use per-row (or a wrapping array field) for repeating entities.

## Tier — pick by complexity and stakes

Tiers trade cost/speed against reasoning quality.

| Situation | Tier |
|---|---|
| Clean, simple, well-structured docs; volume/cost sensitive | Cost-effective |
| Most real-world, mixed-complexity documents | **Agentic — the default**; start here |
| Dense tables, intricate structure, high-stakes accuracy | Agentic-plus (highest reasoning) |

Start at the recommended default tier. Move *down* only after confirming accuracy
holds on your documents; move *up* when you see reasoning failures on complex
layouts. Do not pay for the top tier on documents the default handles.

A separate **parse tier** governs how aggressively the document is parsed before
extraction. Raise it for visually complex source PDFs (scans, multi-column,
heavy tables); the default suffices for clean digital text.

## System prompt — global guidance only

Use the system prompt for instructions that span the whole extraction:
disambiguation rules ("if multiple fiscal years appear, prefer the most recent"),
global formatting conventions, domain context. Keep **field-specific** guidance
in the field's description instead — do not duplicate it here.

## Page range

When only part of a large document is relevant, restrict to a page range to cut
cost and latency and reduce off-target extractions. The platform accepts a
compact range expression (single pages and spans). Look up the exact syntax live;
the judgment is simply: scope to the pages that matter when you know them.

## Context window / multi-page reasoning

There is a tension between giving the model enough surrounding pages to reason
across boundaries and exhaustively covering very long, dense lists. For
cross-page fields (a total that depends on earlier pages), favor more context;
for long repeating lists, favor coverage and consider per-page/per-row targets or
splitting (see schema-design.md).

## Citations, reasoning, confidence

These add traceability at the cost of extra processing time/latency:

- **Citations** — link each extracted value back to where it was found in the
  source. Enable when you need auditability, human review, or to debug *why* a
  value was picked. The premier tool for diagnosing wrong extractions.
- **Reasoning** — surfaces the model's rationale for a value; useful while
  tuning schema/prompts.
- **Confidence scores** — per-field reliability signal; use to gate
  low-confidence values for human review in a pipeline.

Leave these off for steady-state high-throughput runs where you have already
validated quality; turn them on for development, audited workflows, and
triaging failures.

## Tuning loop

1. Lock the schema and pick a target matching document shape.
2. Run on representative docs at the default tier with citations on.
3. Inspect misses via citations/reasoning.
4. Fix in this order: field descriptions → system prompt → target → tier.
   Changing the tier first hides schema problems you will pay for at scale.
