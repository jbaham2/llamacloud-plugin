---
description: Router for the LlamaCloud / LlamaIndex plugin — describe what you're building and get pointed to the right product, skill, and next step.
argument-hint: [what you're trying to build, e.g. "extract fields from invoices"]
allowed-tools: ["mcp__plugin_llamacloud_docs__search_docs", "mcp__plugin_llamacloud_docs__read_doc"]
---

The user invoked the LlamaCloud router. Their goal: **$ARGUMENTS**

Route them to the right product and skill. Do this:

1. If `$ARGUMENTS` is empty, ask one concise question: what are they building (parsing, structured
   extraction, classification, splitting, spreadsheets, RAG/retrieval, OSS framework, agentic
   workflows, deployment, or self-hosting)?

2. Map the goal to the owning product/skill using this table. Name the product, then let the
   matching skill take over (the skills auto-trigger on their phrases):

   | Goal | Product | Skill |
   |---|---|---|
   | Clean text/markdown from documents | Parse | `llamacloud-parse` |
   | Typed fields matching a schema | Extract | `llamacloud-extract` |
   | Managed RAG / hosted retrieval | Cloud Index | `llamacloud-index` |
   | Route docs by type | Classify | `llamacloud-classify` |
   | Segment one combined file | Split | `llamacloud-split` |
   | Reason over messy spreadsheets | Sheets | `llamacloud-sheets` |
   | Local/offline parsing, no key | LiteParse | `liteparse` |
   | Self-host the platform (BYOC/Helm) | — | `llamacloud-self-hosting` |
   | Build RAG yourself in code (OSS) | — | `llamaindex-framework` |
   | Event-driven agentic workflows | — | `llamaindex-workflows` |
   | Deploy/serve a workflow or agent | — | `llamaindex-deploy` |
   | Find an example/recipe | — | `llamaindex-cookbooks` |

3. If the goal spans products, give the sequence (e.g. Split → Classify → Extract; or Parse → Cloud
   Index) and start with the first.

4. If the user has not set up access yet, point them to `/llamacloud-setup` first.

5. Keep stale-prone facts out of your answer — for current package names, signatures, pricing, or
   limits, use `search_docs` / `read_doc` rather than asserting from memory. Encode the version
   posture: Parse v2 / Extract v2 are the GA default; `/v1/` is legacy; Cloud Index v2 and
   Classify/Split/Sheets are beta.

Be brief: name the product, name the skill that will help, and give the immediate next step.
