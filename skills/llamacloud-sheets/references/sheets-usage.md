# Sheets — usage judgment

Durable judgment for LlamaCloud Sheets (beta): when it fits vs Extract, how to work with its output,
and the gotchas. Signatures, package names, status enums, and limit numbers are fetched live (MCP or
`index.md` WebFetch) — this file is *which approach, when, and why*.

## Sheets vs Extract — the boundary

Both can touch a spreadsheet. The deciding question is **what shape do you want back**.

| You want… | Use | Why |
|---|---|---|
| The table(s) **as they exist**, normalized into rows | **Sheets** | Returns the region's own columns/rows; no schema to define |
| A **specific set of typed fields** you define ("total, vendor, date") | **Extract** | You declare a schema; Extract maps content to it, with grounding |
| Rows you'll **aggregate / join / filter** downstream | **Sheets** | Parquet → Pandas/Polars/DuckDB, types preserved |
| One value per document, or fields spread across prose + tables | **Extract** | Extract parses internally and reasons across the whole doc |
| Source is a **PDF / scan / image**, not a workbook | **Extract** / **Parse** | Sheets is for spreadsheet grids |
| You don't know where the table starts (banner, blank rows, stacked tables) | **Sheets** | Region detection finds and isolates each grid |

Rules of thumb:

- **"Give me the table"** → Sheets. **"Give me these fields"** → Extract.
- A spreadsheet with several stacked tables, where you want *each table* → Sheets (one region each).
  A spreadsheet where you want *three numbers off it* → Extract is simpler.
- Don't define a wide Extract schema just to recover a table's columns — that's reinventing Sheets
  and you lose the normalized typing Sheets gives for free.
- Conversely, don't post-process Sheets Parquet with your own LLM to pull typed fields when Extract
  would do it in one call with grounding.

## Working with the output

### Regions
- A job can return **many regions per sheet** and **many sheets**. Don't assume one table. Iterate
  regions; use each region's `location` range and generated `title`/`description` to pick the one(s)
  you want.
- Filter by `sheet_name` + `location` (or the generated `title`) rather than positional index — the
  region order is not a stable contract.
- When metadata generation is enabled, the `title`/`description` are model-generated summaries. Good
  for routing/selection; don't treat them as authoritative column semantics — read the data.

### Table data vs cell metadata
- **Default to table-data Parquet.** It's the normalized rows with types preserved; load with
  `pd.read_parquet`, Polars, or DuckDB `read_parquet()` and start querying.
- Download **cell metadata only when you need structure the normalized table can't express** —
  e.g. detecting a header by bold formatting, finding emphasized cells, or reading currency/percent
  formatting to disambiguate a numeric column. It's a *separate* download; skip it otherwise (extra
  payload, extra latency).
- Cell metadata is for *inferring* structure, not for re-deriving the table — the table data already
  did the normalization.

### Reasoning over rows
- Parquet is columnar and self-describing: no schema declaration needed downstream.
- For multi-region/multi-sheet jobs, load each region into its own frame and join/concat explicitly;
  don't blindly stack regions that have different columns.
- DuckDB is convenient for ad-hoc joins/aggregations across several region files without loading
  everything into memory.

## Status handling

- Terminal states: `SUCCESS`, `PARTIAL_SUCCESS`, `ERROR`, `FAILURE`. Transient: `PENDING`.
- **`PARTIAL_SUCCESS` is not a failure** — some regions succeeded. Process the regions you got and
  inspect the `errors` array to decide whether to retry the rest. Treating it as all-or-nothing
  silently drops usable data.
- **Poll, don't block-assume.** Results are async and not inline; there is no synchronous result.

## Beta gotchas & anti-patterns

- **Throttle (~1 req/s on job create):** batch uploads with backoff; a tight loop of `create` calls
  will hit the limit. Confirm the current rate from docs.
- **Size cap (~100 cols / ~10k rows per sheet):** split oversized sheets *before* upload; don't
  expect Sheets to chunk a giant sheet for you.
- **Presigned download URLs expire.** Fetch them from a fresh job result at download time; don't
  cache and reuse a stale URL.
- **Beta namespace.** The API lives under `beta.sheets` — names and shapes can change release to
  release. Verify signatures live (MCP) rather than pinning to a remembered method name.
- **Anti-pattern:** chaining Parse → custom code to rebuild a spreadsheet's tables. If the source is
  a spreadsheet, Sheets already does region detection + normalization; hand-rolling range parsing
  loses that.
- **Anti-pattern:** using Sheets to get a handful of typed fields. That's `llamacloud-extract`.

## Fetching facts

Verify live before relying on anything drift-prone (package name, method/param signatures, status
enum, throttle/size numbers, base URL):
- MCP: `mcp__plugin_llamacloud_docs__search_docs`, `..._read_doc`, `..._grep_docs`.
- Fallback: append `index.md` to a `https://developers.llamaindex.ai/llamaparse/sheets/...` URL and
  `WebFetch`.
