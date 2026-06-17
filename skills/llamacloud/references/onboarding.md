# Onboarding — setup checklist, wiring, security, pricing model

Durable setup judgment. Exact package names, versions, and pricing numbers are **fetched live** —
this file gives the sequence, the decisions, and the failure modes.

## Setup checklist

1. **Create an account** in the LlamaCloud console (NA or EU — see region decision below).
2. **Generate an API key** scoped to the right project/organization. One key works across all
   products (Parse, Extract, Classify, Split, Sheets, Cloud Index).
3. **Export the key** as `LLAMA_CLOUD_API_KEY` in the environment. Never hardcode it in source,
   notebooks, or chat. Use a secrets manager / `.env` (gitignored) in real projects.
4. **Choose and pin the region** (one-time — see below).
5. **Install the SDK** — confirm the current package name/version from the docs first.
6. **Smoke-test** with the smallest operation in the target product, then hand off to that skill.

## Region decision (one-time, non-reversible)

| | NA | EU |
|---|---|---|
| API base host | `api.cloud.llamaindex.ai` | `api.cloud.eu.llamaindex.ai` |
| Console host | `cloud.llamaindex.ai` | `cloud.eu.llamaindex.ai` |
| New features | first | may lag |
| Data residency | US | data stored & processed in EU |
| Migrate later? | **No** | **No** |

Decision drivers: GDPR / EU data-residency obligations push to EU; otherwise NA for earliest
features. Pick EU from anywhere if residency requires it. **Data cannot move between regions after
the choice** — treat it as an architecture decision, not a setting.

The SDK targets NA by default. For EU, point the client at the EU base URL (the exact constructor
arg / `base_url` differs by SDK version — fetch it live).

## Env + region wiring pattern

The durable pattern (illustrative — confirm exact symbols live):

- **Python:** read `os.environ["LLAMA_CLOUD_API_KEY"]`; pass `api_key=` (and a `base_url`/region arg
  for EU) when constructing the service client (`LlamaParse`, `LlamaExtract`, …).
- **TypeScript:** read `process.env.LLAMA_CLOUD_API_KEY`; pass it (and EU base URL when applicable)
  into the client constructor.

Keep the key in the environment and let the SDK pick it up; avoid threading raw strings through code.

## Key security

- Treat the key like a password: secrets manager or gitignored `.env`, never committed.
- Use **separate keys per environment** (dev/stage/prod) and per project where possible, so one can
  be rotated or revoked without breaking everything.
- **Rotate** on suspected exposure; revoke unused keys in the console.
- Do not log the key or include it in error reports / screenshots.

## Pricing / credits mental model (numbers are live facts)

- Billing is **credit-based** and **unified** across products; different operations and Parse tiers
  consume different credit amounts.
- The durable judgment: **higher-quality / agentic tiers cost more credits per page**; cost scales
  with pages processed and the tier chosen. Use Parse's tier selection (and any cost-optimizer
  option) to trade quality for cost deliberately — see `llamacloud-parse`.
- A free allotment typically exists for evaluation. **Fetch current credit costs, tier multipliers,
  and free-tier limits live** from `/llamaparse/general/pricing/` — never quote remembered numbers.

## Common first-run failures

| Symptom | Likely cause |
|---|---|
| 401 / auth error | `LLAMA_CLOUD_API_KEY` unset, wrong env, or key from the other region |
| Works in NA, 404/empty in EU | client still pointed at NA base URL; set the EU base URL |
| Import error / unknown class | wrong/old package installed — reconfirm package name & version live |
| Unexpected credit burn | high-tier Parse on large docs — review tier/cost-optimizer settings |
| Data-residency audit fail | account created in NA when EU was required — not migratable; recreate in EU |

## Hand-off

Once the key verifies and the region is set, route to the product skill (`llamacloud-parse`,
`llamacloud-extract`, `llamacloud-index`, …) for the actual build. This skill's job is done once the
user is authenticated, regioned, and pointed at the right product.
