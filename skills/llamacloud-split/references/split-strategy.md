# Split strategy: when, boundaries, sequencing, failure modes

Durable judgment for LlamaCloud Split. Exact endpoints, field names, and limits live in the
docs — fetch them (see SKILL.md "Fetching facts"). This file is about *decisions*.

## When splitting helps vs. when it is the wrong tool

Split earns its keep only when one file genuinely contains **multiple distinct documents**
and you need the boundaries between them.

| Situation | Use Split? | Instead |
|---|---|---|
| One scanned PDF = a stack of separate invoices/contracts/forms | Yes | — |
| Loan/closing package, medical bundle, batch fax, mailroom scan | Yes | — |
| Single document, "what kind is it?" | No | `llamacloud-classify` |
| Single document, "pull the fields" | No | `llamacloud-extract` |
| Single document, "give me the text" | No | `llamacloud-parse` |
| Already-separate files, one per document | No | Classify/Extract each directly |
| Split *within* one document by chapter/section for RAG chunking | No | This is retrieval chunking, not document demux — use the indexing/parse path (`llamacloud-index`) |

Rule of thumb: if slicing the file at the proposed cut points would yield pieces that are
each a **standalone document a human would file separately**, Split is right. If the pieces
are just *sections of one document*, it is not.

## The Split-vs-Classify boundary (the confusing one)

Split internally classifies pages to decide where to cut, so the two feel overlapping. Keep
them distinct by what they *produce*:

- **Split** answers **WHERE** — it returns page ranges that carve one combined file into
  contiguous sub-documents. Output: segments `(category, page_start, page_end, confidence)`.
- **Classify** answers **WHAT** — given an already-separated document, it returns a type
  label. Output: a label (+ confidence) for the whole document.

Practical implications:
- Split's per-segment category is a *byproduct* of boundary detection, tuned for "does this
  page belong with the previous page?" It is good enough for routing simple bundles.
- When you need a richer or different taxonomy than the split categories, a routing decision
  with its own rules, or a confidence check on the whole sub-document, run Classify on each
  segment *after* splitting. Do not try to make Split's categories serve double duty as the
  authoritative document type when the taxonomies differ.

## Sequencing with Classify and Extract

Canonical multi-document pipeline:

1. **Parse** (optional) — needed only if a downstream step wants markdown/text rather than
   the raw file. Split itself operates on the uploaded file.
2. **Split** — produce page ranges. This is the demux.
3. Slice the original file by the returned page ranges into per-document inputs.
4. **Classify** (conditional) — confirm/refine each sub-document's type. Skip when Split's
   categories are already the routing taxonomy and confidence is high.
5. **Extract** — run the schema matched to each document's (now known) type.

Decision: **Split → Extract directly** vs. **Split → Classify → Extract**

| Add a Classify pass when… | Skip Classify when… |
|---|---|
| You need a finer/different taxonomy than the split categories | Split categories already match your routing labels |
| Routing needs natural-language rules beyond category descriptions | Bundle is homogeneous and well-separated |
| Segments come back low-confidence and need a second opinion | Split confidence is uniformly high |
| Downstream extract schema is chosen by a label Split doesn't emit | One extract schema covers all segments |

## Confidence handling

Each segment carries a confidence rating (high / medium / low). Treat it as a routing signal,
not ground truth:

- **High** — proceed automatically to Extract with the matched schema.
- **Medium** — proceed but flag for sampling/QA; consider a Classify pass to confirm type.
- **Low** — route to human review or a Classify pass before extraction. Low confidence often
  clusters at true boundaries (cover pages, transitions) — see failure modes.

## Failure modes and mitigations

- **Vague category descriptions → mis-assigned pages → wrong cut points.** Descriptions are
  the only steering wheel. Write them to be discriminative ("a vendor bill *with line items
  and a total due*"), not generic ("a financial document"). Iterate on descriptions before
  blaming the model.
- **Adjacent same-type documents merge into one segment.** Two consecutive invoices with no
  distinguishing cover page may be grouped together because consecutive same-category pages
  collapse into one segment. Mitigate with categories that capture per-document markers, or
  post-process page ranges with a rule (e.g. detect "Invoice #" resets).
- **Cover/blank/divider pages land in the wrong segment or none.** Decide up front whether to
  allow an "uncategorized" bucket; without it, ambiguous pages get force-fit into a neighbor.
- **Off-by-one page ranges when slicing.** Confirm whether returned page numbers are 0- or
  1-indexed and whether the end page is inclusive *from the live docs* before cutting the
  file — getting this wrong silently shifts every downstream document by a page.
- **Beta churn.** Field names and job lifecycle can change between releases. Pin behavior with
  a smoke test over a known bundle and re-fetch the doc shapes after upgrades rather than
  assuming the response schema held.
- **Treating Split as chunking.** Splitting one cohesive document into RAG chunks is not what
  Split does; it segments multi-document files. For retrieval chunking use the index/parse
  path (`llamacloud-index`).
