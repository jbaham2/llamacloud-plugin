# Classification design judgment

Durable judgment for designing a Classify-based router: taxonomy design, mode selection,
confidence gating, the Classify-vs-Split boundary, and failure modes. Facts that drift
(package names, signatures, credit costs, mode enum spelling) are not here — look those up
live per the SKILL's "Fetching facts" section.

## Taxonomy design

A good category set is **mutually exclusive and collectively exhaustive (MECE)**.

- **Mutually exclusive** — no document should plausibly belong to two categories. When two
  types overlap (invoice vs. statement, contract vs. amendment), the fix is in the rule
  *descriptions*: name the feature that separates them, not just the features they share.
- **Collectively exhaustive** — every document the system will see should have a home.
  Always include an explicit **catch-all** category (`other`, `unknown`). Without it, the
  model is forced to assign the least-wrong real type, which silently corrupts downstream
  routing.

**Granularity.** Match category granularity to what downstream actually branches on. If
invoices and credit notes go to the *same* extraction schema, don't split them — one
`financial-document` type is simpler and more robust. Add a distinction only when a
downstream step treats the two differently.

**Multi-type documents.** Classify assigns *one* type per file. A document that is
genuinely two kinds at once (a cover letter stapled to a form) is a modeling smell: either
(a) it's actually a concatenated bundle → split first (see below), or (b) define a category
for the *dominant/governing* type and let downstream handle the rest.

## Mode selection (decision)

Choose mode by **where the discriminating signal lives**, then let cost break ties.

| Situation                                                       | Mode        | Why                                              |
| -------------------------------------------------------------- | ----------- | ------------------------------------------------ |
| Categories separable from text, labels, and in-text layout     | Fast        | Cheaper; text is sufficient                      |
| Handwriting, scanned/photographed pages, stamps, signatures    | Multimodal  | Text extraction loses the signal                 |
| Distinguishing feature is a chart, logo, form layout, or photo | Multimodal  | Visual structure is the discriminator            |
| Unsure                                                          | Fast first  | Measure accuracy on a sample; upgrade only if it fails |

Multimodal costs more per page. Do not pay for vision globally if only a minority of files
need it — if you can cheaply pre-sort (e.g. born-digital vs. scanned), route only the
scanned subset to Multimodal.

## Confidence gating (the core beta pattern)

Classify returns a `confidence` in 0.0–1.0 per file. Because Classify is **beta** and no
classifier is perfect, **never auto-route on the predicted type alone.** Gate on confidence:

1. **Calibrate a threshold empirically.** Run Classify on a labeled sample, plot accuracy
   vs. confidence, and pick the threshold where auto-routing meets your error budget. Do
   not hardcode a guessed value like 0.8.
2. **Three lanes, not two:**
   - High confidence + a real category → auto-route to the type's pipeline.
   - Low confidence, or the catch-all category → human-review / default queue.
   - Errors / job failure → retry, then dead-letter; never drop silently.
3. **Keep a fallback path.** Downstream must degrade gracefully (default pipeline, manual
   triage) when Classify is unavailable or unsure. Do not make Classify a hard SLA
   dependency with no escape hatch.
4. **Log type + confidence + reasoning** for every decision. This is your audit trail and
   your data for re-calibrating the threshold and sharpening rules over time.

## Classify vs. Split (the sharp boundary)

This is the most common modeling mistake.

- **Classify** answers *"what type is this whole file?"* — one label per file.
- **Split** (`llamacloud-split`) answers *"where does one sub-document end and the next
  begin?"* — it carves a single concatenated file into separate documents.

If a file contains **multiple documents** (a scanned stack, a merged PDF of unrelated
forms), Classify will mislabel it — it must produce one type for the whole thing.
**Correct order: Split first, then Classify each resulting piece.** Reach for Split, not a
cleverer rule, whenever the unit you want to label is smaller than the file.

## Pipeline composition

Classify is rarely terminal. Typical chains:

- `llamacloud-split` → `llamacloud-classify` → branch → `llamacloud-extract`
  (segment a bundle, type each piece, extract with the schema chosen by type).
- `llamacloud-classify` → `llamacloud-parse` (route, then parse with a type-appropriate
  preset).

Classify is the cheap routing gate that lets you spend expensive type-specific processing
only where it applies.

## Failure modes and anti-patterns

| Symptom                                              | Likely cause                                        | Fix                                                            |
| ---------------------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------- |
| Two types constantly confused                        | Descriptions share features, lack a contrast        | Add the distinguishing feature; contrast against the look-alike |
| Off-distribution docs forced into a wrong type       | No catch-all category                               | Add an explicit `other`/`unknown` category                    |
| Multi-document file gets one wrong label             | Used Classify where Split was needed                | Split first, then Classify each piece                         |
| Handwritten/scanned docs misclassified on Fast       | Signal is visual, mode is text-only                 | Switch (or pre-route) those files to Multimodal               |
| Confident-but-wrong routing in prod                  | Auto-routed on type without a confidence gate       | Add the confidence threshold + review lane                    |
| Categories balloon, accuracy drops                   | Over-granular taxonomy beyond what downstream needs | Collapse categories that share a downstream path              |
| Pipeline breaks when Classify changes                | Treated beta product as a hard dependency           | Keep a fallback path; pin and re-test SDK versions            |

**Debugging loop.** When accuracy is low, do not blindly rewrite all rules. Read the
`reasoning` field on the misclassified items — it states *why* the model chose the wrong
type — and sharpen only the description that lost the comparison. Re-run on the sample.
Repeat.
