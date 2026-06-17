# LiteParse vs cloud LlamaParse — decision depth

Durable judgment for choosing the local OSS parser (LiteParse) vs the hosted cloud
service (LlamaParse / Parse v2), and for picking a LiteParse usage mode. Verify
package names, signatures, flags, and pricing live (see SKILL.md "Fetching facts").

## The core trade-off

LiteParse maximizes control, privacy, and cost predictability. Cloud LlamaParse
maximizes parse quality on hard documents via models and managed infrastructure.
They are complementary, not redundant: a common pattern is local-by-default,
escalate-on-difficulty.

| Dimension            | LiteParse (local/OSS)                          | Cloud LlamaParse (`llamacloud-parse`)            |
|----------------------|------------------------------------------------|--------------------------------------------------|
| API key              | None                                           | Required (`LLAMA_CLOUD_API_KEY`)                 |
| Where it runs        | Your machine / browser / private network       | LlamaCloud (data leaves the boundary)            |
| Models / LLMs        | None (deterministic engine + Tesseract OCR)    | Model-driven, optional agentic modes             |
| Privacy / air-gap    | Strong — nothing leaves                         | Data sent to cloud (gating concern)              |
| Cost model           | Compute you already own; no per-page credits   | Per-page / credit-based                          |
| Latency              | Local, no network round-trip                    | Network + queue                                  |
| Parse quality ceiling| Good on clean/moderate docs                     | Highest on dense tables, complex layouts, charts |
| Scaling              | Bounded by local hardware                       | Managed, elastic                                 |
| Offline              | Yes                                             | No                                               |

## When LiteParse is the right call

- **Regulated / sensitive data** that cannot leave the host or network (air-gapped).
- **Inner dev loop** — iterating on a parsing/RAG pipeline where free, fast,
  deterministic local runs beat paying per page.
- **High-volume, simpler documents** — digital PDFs, Office files, clean scans where
  layout text + OCR + bounding boxes are sufficient.
- **Client-side / edge** — must run in the browser (WASM) with no backend.
- **Need raw geometry** — per-line bounding boxes for citations, highlighting, or
  custom layout logic.

## When to escalate to cloud LlamaParse

- Dense or merged-cell **tables**, complex multi-column layouts, rotated/skewed pages.
- **Handwriting**, low-quality scans, or figure/chart understanding.
- Need **agentic / premium** parse modes or the highest fidelity at scale.
- Don't want to manage compute/OCR infrastructure.

Hybrid: route easy pages through LiteParse, send pages that fail a quality bar (low
text yield, detected complex tables) to cloud. Keep the same downstream schema so
either source feeds the same pipeline.

## Anti-patterns

- Reaching for the cloud for **simple digital PDFs** — wastes credits and adds latency.
- Using LiteParse for **complex tables/handwriting** and shipping degraded extraction.
- Sending **regulated data** to the cloud when local parsing was sufficient.
- Re-constructing a parser instance per file in a hot loop — construct once, reuse;
  if many clients need it, use **server mode**.

## Picking a LiteParse usage mode

| Mode    | Use when                                                      | Trade-offs                                          |
|---------|---------------------------------------------------------------|-----------------------------------------------------|
| Library | Embedding parsing inside a Python/TS/Rust app or service       | Best integration; construct once and reuse instance |
| CLI     | One-off, scripted, or batch parsing from the shell             | No code; less control over the result object        |
| Server  | Many clients / multiple languages share one parser process     | One init cost amortized; adds a process to operate  |
| WASM    | Parsing must run in the browser, fully client-side             | No server/data egress; bounded by browser resources |

Sequencing guidance:
1. Prototype with **library or CLI** locally to validate output quality on real docs.
2. If init/startup cost dominates under concurrency, move to **server** mode.
3. If documents must never reach a backend, use **WASM**.
4. If a quality gap appears on hard documents, add a cloud-escalation path
   (`llamacloud-parse`) rather than abandoning the local default.

## OCR notes (gotchas)

- Scanned/image-only pages need OCR; built-in Tesseract covers common cases.
- OCR language must match the document, or accuracy drops — set it explicitly.
- A custom/external OCR server can be pointed to for better quality or languages
  Tesseract handles poorly. Verify the exact config parameter name live.
- Render `dpi` affects OCR accuracy and screenshot quality vs speed/memory — raise it
  for small text, lower it for throughput.

## Where LiteParse stops

LiteParse produces parsed text, bounding boxes, and page images. It does not do
typed schema extraction (→ `llamacloud-extract`) or managed indexing/RAG retrieval
(→ `llamacloud-index`). Treat its output as the parse stage feeding those.
