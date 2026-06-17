# Extraction schema design — durable rules

The schema *is* the extraction program. Field names and descriptions are the
primary lever on output quality; the LLM reads them at extraction time. Spend
effort here before reaching for tier/mode knobs.

## Structural rules (will not change)

- **Root must be an object.** A schema cannot have an array (or scalar) at the
  top level. To extract a variable number of entities, wrap them in a field:
  `{ "line_items": [ ... ] }`, not a bare list.
- **Keep nesting shallow.** The platform allows deep nesting, but treat 3–4
  levels as the practical ceiling. Deep trees confuse the model and make
  validation/debugging painful. Flatten when you can.
- **There are hard caps** on nesting depth, total property count, and total
  string length of the schema. They are large enough that well-designed schemas
  never hit them — but auto-generated or machine-translated schemas can. Look up
  the current numbers live (see SKILL.md "Fetching facts") rather than memorizing
  them; if you are near any cap, the schema is almost certainly over-modeled.

## Field naming

- Use clear, stable names that match how you will store/query the data
  (`invoice_number`, `total_amount`), because they become the JSON keys in the
  result. Renaming later means rewriting downstream consumers.
- Avoid presentational or ambiguous names. The name plus the description must let
  a reader (and the model) know exactly what value belongs there.

## Descriptions do the work

- Every non-obvious field needs a description stating: what the value is,
  acceptable format, and an example. This is where you encode "MM/DD/YYYY",
  "ISO currency code", "exclude tax", etc.
- Field-level instructions belong in descriptions. *Global* instructions
  ("prefer the most recent fiscal year") belong in the system prompt, not
  repeated per field — see options-and-tiers.md.

## What JSON Schema features are honored — and what is not

Several standard JSON Schema constructs are **not** enforced by extraction:
defaults, format validators, and similar constraints are ignored. Do not rely on
them for correctness. Instead:

- Encode the requirement in the field **description** (the model will follow it),
  and/or
- **Post-process** the result in your own code (coerce types, validate formats,
  apply defaults).

Optional-ness *is* honored. Mark a field optional rather than required when the
value may be absent:

- Pydantic: `Optional[...]` / a default of `None`.
- Zod: `.nullable()` / `.optional()`.
- Raw JSON Schema: omit from `required`, or use `anyOf` with a `null` type.

## Optional vs. required — the booleans/ints gotcha

Make **boolean and integer fields optional** unless the value is truly always
present. A required `bool`/`int` invites the model to emit a misleading default
(`false`/`0`) when the document simply does not contain the value, silently
corrupting downstream logic. Optional + null distinguishes "absent" from "zero".

## Authoring path: Pydantic / Zod / JSON Schema

- **Python → Pydantic** is the recommended authoring surface; **TypeScript → Zod**
  is its counterpart. Both compile down to JSON Schema for the API, so you can
  also hand-write JSON Schema directly when you need full control.
- Prefer the typed model in your language: you get IDE help, reuse the same model
  to validate the extracted result, and avoid hand-maintaining JSON.
- For exact class/decorator/parameter names, look them up live via MCP rather
  than trusting any snippet — the SDK surface has shifted between releases.

## Iteration strategy (the real best practice)

1. **Start minimal.** Model only the fields you actually need. Every extra field
   is a chance for the model to hallucinate or slow down.
2. **Auto-generated schemas are a draft, not a deliverable.** Schema inference
   tends to over-produce fields. Review and prune before production.
3. **Test on representative documents**, including messy/edge ones. Read the
   misses, then fix them by sharpening descriptions — not by adding fields.
4. **Validate the output** against your typed model in code; treat validation
   failures as schema-quality signals.
5. **Split and merge when too big.** If one schema is unwieldy or brushes the
   caps, extract field subsets in separate passes and merge results in your code.
   This is also a latency/accuracy win for very large documents.

## Anti-patterns

- One giant schema with dozens of optional fields "just in case."
- Deeply nested mirrors of the document's visual layout.
- Relying on JSON Schema `default`/`format` for correctness.
- Required booleans/integers for values that may be absent.
- Shipping an inferred schema unreviewed.
- Vague field names with no descriptions.
