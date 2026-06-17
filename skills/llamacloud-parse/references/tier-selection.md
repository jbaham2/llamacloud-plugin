# Tier selection

Pick a parse tier by **document complexity first**, then let cost follow. The live tiers, from
lowest to highest capability, are **Fast → Cost Effective → Agentic → Agentic Plus**, with the
**Cost Optimizer** as a per-page router for mixed documents. (Tier names and identifiers can
change — confirm current ones via the MCP / live docs before hard-coding them.)

## Decision table

| Document profile | Tier | Why |
|---|---|---|
| Plain text, high volume, no tables, no markdown needed | **Fast** | Cheapest, fastest, basic text only. No markdown, no custom prompt. |
| Mostly prose with simple/straightforward tables | **Cost Effective** | Cleaner markdown than Fast at moderate cost; trades deep layout analysis for affordability. |
| Real-world PDFs: tables, scanned pages, multi-column, charts | **Agentic** | The recommended **default** for documents needing real parsing without extreme complexity. |
| Dense financial reports, scientific papers, equations, mission-critical accuracy | **Agentic Plus** | Premium; chart parsing on by default; for the most demanding layouts where errors are costly. |
| One document whose pages vary (some prose, some dense tables/charts) | **Cost Optimizer** (on Agentic+) | Routes each page to the appropriate tier in parallel — premium quality on hard pages, cheaper processing on easy ones, without a latency penalty. |

## How to reason about it

1. **Start from the hardest content in the document, not the average.** A 90%-prose report with
   three critical financial tables is an Agentic/Agentic Plus job, because the tables are what you
   actually need correct.
2. **Default to Agentic** when unsure and the document is a normal real-world PDF. It covers the
   majority of cases.
3. **Step down to Cost Effective / Fast only when you can name what you're giving up** — Fast is
   text-only with no markdown and no custom prompt; Cost Effective skips deep layout analysis.
4. **Step up to Agentic Plus** for charts-as-data, equations, dense multi-column financial/
   scientific layouts, or when a parsing error has downstream consequences.

## Capability constraints that follow from the tier

Tier choice is not just a price dial — it gates features. The ones that surprise people:

- **No markdown on the lowest (Fast) tier.** Requesting markdown output or markdown expand values
  on Fast returns a validation error. If you need markdown, you need Cost Effective or higher.
- **Chart parsing is off on Fast** and on by default on Agentic Plus.
- **Custom natural-language prompts** are unavailable on Fast (supported on Cost Effective and up).
- **Granular bounding boxes** are unavailable on the Fast tier.

So "which tier" and "which output/feature" are coupled decisions: choose the tier that *can*
produce the output you need, then confirm the feature is enabled for that tier.

## Cost Optimizer — when and when not

- **Use it** for heterogeneous documents where most pages are easy prose but some are dense. It
  parallelizes, so there is no latency cost for the routing.
- **Skip it** when you need exact reproducibility / benchmarking, or when the whole document is
  uniformly complex (just pick Agentic Plus) or uniformly simple (just pick the cheap tier).

## Anti-patterns

- Choosing a tier by price and then being surprised that markdown or tables are wrong — complexity
  drives quality; underspending on a complex document wastes the whole parse.
- Defaulting everything to Agentic Plus "to be safe" on simple high-volume text — overpaying with
  no quality gain; Fast or Cost Effective is correct there.
- Treating Fast as a cheaper Agentic — it is a different capability tier (no markdown), not a
  discount.
