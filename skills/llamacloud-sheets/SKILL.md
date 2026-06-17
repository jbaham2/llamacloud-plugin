---
name: llamacloud-sheets
description: This skill should be used when the user wants to "extract from spreadsheets", "use LlamaCloud Sheets", "reason over spreadsheet rows", or "pull tables from messy Excel/CSV" — detecting table regions in disorganized workbooks and turning them into normalized rows. Owns spreadsheet table extraction and its output format; defers typed field extraction from documents to `llamacloud-extract`, raw text to `llamacloud-parse`, and account/key setup to core `llamacloud`.
version: 0.1.0
---

# LlamaCloud Sheets — messy spreadsheets → reasoned-over rows

LlamaCloud Sheets (LlamaSheets) takes a disorganized `.xlsx` / `.csv` — stacked tables, header
banners, merged cells, notes scattered around the grid — and finds the **table regions**, normalizes
each into a clean typed table, and returns the result as **Parquet**. The point is to get from a
human-formatted workbook to rows a program can load and reason over (Pandas, Polars, DuckDB) without
hand-writing range parsing.

**Sheets is beta.** Set that expectation up front: the API surface lives under a `beta` namespace,
throughput and sheet size are capped (see Limits), and signatures can shift. Treat it as production-
viable for batch/offline normalization, not yet as a hardened low-latency path.

## When Sheets is the right tool

Use Sheets when **the source is a spreadsheet** and the deliverable is **rows/tables**, not a fixed
set of typed fields:

- A workbook where the real table doesn't start at A1 — there's a title banner, a logo, blank rows,
  then the grid. Sheets locates the region (e.g. `A4:H230`) for you.
- One sheet holding **several stacked tables** with gaps between them — Sheets returns each as a
  separate region.
- You want to **aggregate / filter / join** the rows afterward (group by, sum, dedupe) — Parquet
  loads straight into Pandas/Polars/DuckDB with types preserved.
- You need **cell-level metadata** (formatting, position, detected type) to infer structure — e.g.
  bold first row = header, currency-formatted column = amount.

Reach for a sibling instead when:

- The source is a **PDF / scan / document**, not a spreadsheet → `llamacloud-parse` (clean text) or
  `llamacloud-extract` (typed fields).
- You want **a specific typed schema** ("invoice_total, vendor, due_date") rather than the table's
  own rows — even from a spreadsheet, that intent is `llamacloud-extract` territory. Sheets gives you
  *the table as it is, normalized*; Extract gives you *fields you defined*. See
  `references/sheets-usage.md` for the full Sheets-vs-Extract boundary.

## The extraction flow

Four async steps; the SDK exposes a high-level method per step (confirm exact names live — see
Fetching facts):

1. **Upload** the spreadsheet → returns a file id.
2. **Create** a Sheets job from the file id, under the SDK's `beta.sheets` namespace.
3. **Poll** the job until status is terminal: `SUCCESS`, `PARTIAL_SUCCESS`, `ERROR`, or `FAILURE`
   (transient state is `PENDING`). `PARTIAL_SUCCESS` means some regions came back and others
   failed — inspect the `errors` array, don't assume all-or-nothing.
4. **Download** results per region — Parquet table data, and optionally a separate cell-metadata
   Parquet — via the presigned URLs the result returns.

Polling is mandatory; results are not inline. Presigned download URLs **expire** — fetch them from a
fresh result rather than caching the URL.

### Job config — the two decisions that matter

Create-job config is optional, and the defaults (parse **all** sheets, **generate** region/sheet
metadata) are sensible. The two knobs worth a conscious choice:

- **Which sheets to parse.** Restrict to the sheets you care about when a workbook has many tabs you
  don't need — it cuts cost and noise versus parsing everything.
- **Whether to generate metadata.** The generated `title`/`description` summaries make region
  selection easy. Turn generation off only when you already know exactly which region you want and
  want the leanest, fastest job.

Look up the exact config field names and defaults live (MCP) before relying on them.

### Sequence (illustrative — fetch exact method names via MCP)

```
# upload the spreadsheet        -> file id
# create a beta sheets job      -> job id   (beta.sheets namespace)
# poll the job's get-status     -> until status is terminal
# fetch the result table        -> presigned Parquet URL(s)
# load the Parquet              -> pandas / polars / DuckDB, reason over rows
```

This is the durable shape — upload → create → poll → download → load. The exact `beta.sheets` method
and parameter names are the most drift-prone part of this surface; **grep them via MCP**
(`mcp__plugin_llamacloud_docs__grep_docs`) rather than copying them from memory. The high-level
Python surface is commonly `llama-cloud-services` and the TS one `@llamaindex/llama-cloud` — confirm
both live before pinning.

## Output: regions and Parquet

A successful job returns, per worksheet:

- **`regions`** — each with a `region_id`, the `sheet_name`, a `location` range (`A2:E11`), and a
  generated `title` / `description` when metadata is enabled.
- **`worksheet_metadata`** — `sheet_name`, `title`, `description` at the sheet level.

For each region you download up to two Parquet files:

- **Table data** — the normalized rows, types preserved. This is what you load and query.
- **Cell metadata** (separate file, separate download) — per-cell position, formatting (font/color/
  border/alignment), and detected type (date/percent/currency). Pull this only when you need to infer
  structure beyond the normalized table.

Parquet is columnar and portable: `pd.read_parquet`, Polars, or a DuckDB `read_parquet()` query all
load it directly with no schema declaration. See `references/sheets-usage.md` for handling the
output well (region selection, multi-table sheets, when cell metadata earns its keep).

## Limits (beta)

- **Input:** spreadsheet workbooks — `.xlsx` and `.csv` grids. For documents that merely *contain*
  tables (PDFs, scans), use `llamacloud-parse` / `llamacloud-extract` instead.
- **Throughput:** job creation is throttled (low single-digit requests/sec — confirm the exact rate
  live) — batch with backoff, don't fan out hundreds of `create` calls at once.
- **Size:** each sheet caps at a fixed column/row limit (whichever hits first; confirm the current
  values live). The cap is per *sheet*, not per workbook. A sheet over the cap is the upstream split
  point: break it into sheets within the limit **before** upload rather than expecting Sheets to
  chunk it for you — don't rely on graceful truncation.

The throttle and the size cap exist; their exact values are drift-prone — fetch them (and the exact
over-limit behavior) from the docs before relying on them.

## Fetching facts (do not hardcode)

Exact package names/versions, method/parameter signatures, base URLs, throttle/size limits, and
status enums **drift** — fetch them live, never from memory:

- **Docs MCP** (wired, no auth): `mcp__plugin_llamacloud_docs__search_docs` for concepts,
  `mcp__plugin_llamacloud_docs__read_doc` for a known page, `mcp__plugin_llamacloud_docs__grep_docs`
  for an exact symbol (e.g. `get_result_table`, `region_id`).
- **Fallback:** append `index.md` to any `https://developers.llamaindex.ai/...` page URL and
  `WebFetch` it.

Anchor page: `/llamaparse/sheets/`. For exact SDK signatures, prefer the MCP over reproducing them.

## Cross-links

- Account, API key, region choice, SDK install → core `llamacloud` (don't re-teach setup here).
- Typed-field extraction from documents (or from a spreadsheet against a schema you define) →
  `llamacloud-extract`.
- Raw clean text/markdown from a document → `llamacloud-parse`.

## Additional resources

- **`references/sheets-usage.md`** — Sheets-vs-Extract decision table, working with the output
  (region selection, multi-table and `PARTIAL_SUCCESS` handling, cell-metadata judgment), and beta
  gotchas / anti-patterns.
