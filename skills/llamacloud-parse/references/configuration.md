# Configuration judgment and retrieval gotchas

This is about *which option category to reach for when* and *what bites you* — not parameter
names. For exact parameter keys and current defaults, use the LlamaCloud MCP or the live
`configuring-parse` / `retrieving-results` docs.

## The four config families — when to touch each

**Input controls.** Point the job at a file or a source URL. Reach for page restriction or a page
cap when you only care about part of a long document (cheaper, faster). Use geometric cropping to
strip *repeating* headers/footers. File-type hints exist for HTML (hide nav/fixed chrome),
spreadsheets (sub-table detection, formula recompute), slides (out-of-bounds content), and images
(camera-photo perspective/lighting correction) — set these only when the default mis-handles your
source.

**Output format.** Default to **markdown** — it is the right input for almost every LLM pipeline.
Choose **JSON typed items** when downstream code needs to walk tables/headings as structured
elements. Choose **spatial/plain text** for forms, CAD, or layout-sensitive multi-column content
where whitespace carries meaning. Per-table spreadsheet export, image assets, and bounding boxes
are opt-in for specialized needs (analysis, highlighting, grounding).

**Processing options.** Cost Optimizer (see `tier-selection.md`), specialized chart parsing
(charts → structured data), custom natural-language prompts (steer focus / preserve formats / give
document context), content-skipping (watermarks, hidden / white-on-white text, low-quality image
OCR), OCR language hints (affect image-based text only, not native PDF text), and table tuning.
Add these one at a time to solve a *specific* observed defect — not preemptively.

**Result retrieval shape.** Covered below.

## Configuration gotchas

- **Any option change busts the cache.** A re-parse with even one changed option is a fresh,
  billable parse. Keep options stable across a corpus; only disable caching deliberately for
  benchmarking.
- **Crop box is geometric, not a content filter.** It removes a fixed rectangle from every page.
  If the chrome you want gone moves between pages, use content-ignore / skipping rules instead.
- **Custom prompt and chart parsing depend on tier.** Both are unavailable on the Fast tier.
  Decide the tier and the feature together (see `tier-selection.md`).
- **OCR language hints only affect image-based text.** Native PDF text is read directly and is
  unaffected — setting a language won't fix a structure problem.
- **Formula recompute is a cost, not a default.** Enable spreadsheet formula recomputation only
  when the sheet was never recalculated or holds placeholder values; it slows formula-heavy sheets.
- **Webhooks must be HTTPS with auth in headers, not query params.** Never expose a webhook URL to
  untrusted users.

## Retrieving results

Two shapes come back:

- **Content fields** return parsed data inline — plain text, markdown, typed JSON items, and
  metadata. This is what most pipelines consume directly.
- **Metadata fields** return **presigned download URLs** instead of inline content — for large
  payloads you want to stream or defer (markdown, items, xlsx, images).

Judgment:

- **Request the minimum.** Most LLM pipelines retrieve markdown alone. Over-expanding inflates the
  response and the work done.
- **Re-retrieve, don't re-parse.** Fetch a completed job again later with different fields
  requested — no second parse and no second charge. Parse once, decide what you need afterward.
- **For large files, check size first** via the metadata/URL path before committing to a download.

## Retrieval gotchas

- **Fast tier has no markdown.** Asking for markdown (output or expand value) on a Fast-tier job
  returns a validation error — the single most common retrieval failure. Use Cost Effective or
  higher if you need markdown.
- **Presigned URLs expire.** Download promptly or request fresh URLs; don't cache the URL itself.
- **Grounded-items sidecars are automatic.** When granular bounding boxes are enabled, the
  word/line/cell sidecars are included alongside results — you do not separately expand for them.
- **A parse job is the unit, not a single fetch.** The same job ID can serve text, markdown, and
  typed items on demand; pick the field at retrieval time, not parse time.
