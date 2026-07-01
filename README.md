# llamacloud — Claude Code plugin

Makes Claude an expert at **LlamaCloud** (LlamaIndex's document-AI platform) and the **LlamaIndex**
OSS framework + agentic Workflows. The plugin holds *durable judgment* — which product to use when,
schema/RAG/deployment decision tables, failure taxonomies, prod-readiness checklists — and fetches
*stale-prone facts* (API signatures, package names, pricing) live from the official LlamaIndex docs
MCP instead of hardcoding them.

Covers Parse, Extract, Classify, Split, Sheets, and Cloud Index; LiteParse (local/OSS); the
LlamaIndex framework; Workflows; agent deployment (`llamactl`); and BYOC self-hosting. Examples are
provided in both **Python** (`llama-cloud-services`) and **TypeScript**.

> **Version posture:** Parse **v2** and Extract **v2** are the GA default. `/v1/` is legacy /
> migration reference only. Cloud Index **v2** and Classify/Split/Sheets are **beta**. LiteParse is
> local/OSS with no API key.

## Install

This repo is both the plugin and a single-plugin marketplace.

```
/plugin marketplace add jbaham2/llamacloud-plugin
/plugin install llamacloud@llamacloud-marketplace
```

Already cloned the repo? Add it from the checkout instead of GitHub:

```
/plugin marketplace add .        # run from the repo root
/plugin install llamacloud@llamacloud-marketplace
```

Or test locally without installing:

```
claude --plugin-dir /path/to/llamacloud-plugin
```

### Prerequisites

- **LlamaCloud API key** (for running LlamaCloud products, not for this plugin's docs lookups).
  Export it before using the products:
  ```bash
  export LLAMA_CLOUD_API_KEY="llx-..."
  ```
  Run `/llamacloud-setup` for guided onboarding (account, key, region, SDK install, smoke test).
- **Docs MCP:** the plugin wires the official LlamaIndex docs server
  (`https://developers.llamaindex.ai/mcp`, HTTP, **no auth**). Nothing to configure; verify with
  `/mcp` after install. Tools: `search_docs`, `grep_docs`, `read_doc`.

## What's inside

### Skills (auto-trigger on context) — 13

| Skill | Use it for |
|---|---|
| `llamacloud` | Router + onboarding: which product, API key, regions, pricing model |
| `llamacloud-parse` | Parse v2: tiers, configuration, retrieving results |
| `llamacloud-extract` | Extract v2: schema design, options, citations |
| `llamacloud-index` | Cloud Index: managed RAG ingestion + retrieval tuning |
| `llamacloud-classify` | Document classification / intake routing |
| `llamacloud-split` | Segment one concatenated file into sub-documents |
| `llamacloud-sheets` | Extract/reason over messy spreadsheets (beta) |
| `liteparse` | Local/OSS parsing, no API key |
| `llamacloud-self-hosting` | BYOC / Helm / Kubernetes deployment, tuning, monitoring |
| `llamaindex-framework` | OSS RAG: load → index → store → query, component map |
| `llamaindex-workflows` | Event-driven agentic Workflows |
| `llamaindex-deploy` | `llamactl` + serving workflows/agents as APIs |
| `llamaindex-cookbooks` | Find an existing example/recipe for a scenario |

### Agents (delegated) — 5

| Agent | Role | Mode |
|---|---|---|
| `extraction-schema-designer` | Reads sample docs → proposes/validates a LlamaExtract schema | read/write |
| `rag-architect` | Designs a Cloud Index or framework RAG pipeline | read-only analysis |
| `self-hosting-doctor` | Diagnoses a BYOC deployment | **read-only** (never mutates cluster) |
| `cookbook-scout` | Finds matching examples/cookbooks for a scenario | **read-only** |
| `workflow-builder` | Scaffolds a LlamaIndex Workflow / agent deployment | read/write |

### Commands — 2

- `/llamacloud [goal]` — router; points you to the right product, skill, and next step.
- `/llamacloud-setup [product]` — guided onboarding (account, key, region, install, smoke test).

## Usage examples

- "Which LlamaCloud product turns scanned PDFs into clean markdown?" → `llamacloud` routes to `llamacloud-parse`.
- "Design an extraction schema for these invoices" → `extraction-schema-designer`.
- "Should I use Cloud Index or build retrieval myself?" → `rag-architect`.
- "My self-hosted LlamaParse pods are crash-looping" → `self-hosting-doctor` (read-only diagnosis).
- "Scaffold a Workflow that classifies then extracts" → `workflow-builder`.
- "Is there an example for agentic RAG over a Cloud Index?" → `cookbook-scout`.

## Design principles

- **Distill judgment, fetch facts.** `references/` hold durable decisions; signatures/versions/pricing
  are fetched live via the docs MCP or by appending `index.md` to any docs page URL.
- **No duplication.** The docs MCP is wired, not rebuilt (see `VENDORED.md`); each skill owns a
  distinct product with non-overlapping trigger phrases.
- **Least privilege.** Agents declare minimal tools; diagnostic agents are read-only by instruction.

## License

MIT — see `LICENSE`.
