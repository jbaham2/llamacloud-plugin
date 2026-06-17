# Source Tracker

Per-skill source sets are filtered from `llamacloud-doc-roadmap.md` by `proposed pillar`.
This tracker records P1 anchors + status. Facts are fetched live (MCP / `index.md`-append);
only durable judgment is distilled into `references/`.

Status: todo → fetching → distilling → done

| skill | anchor sources (P1) | status | references file |
|---|---|---|---|
| llamacloud | `/llamaparse/`, `/general/api_key/`, `/general/regions/`, `/general/pricing/` | done | product-routing.md, onboarding.md |
| llamacloud-parse | `/parse/`, `/parse/guides/configuring-parse/`, `/parse/guides/tiers/`, `/parse/guides/retrieving-results/` | done | tier-selection.md, configuration.md |
| llamacloud-extract | `/extract/`, `/extract/guides/schema_design/`, `/extract/guides/options/` | done | schema-design.md, options-and-tiers.md |
| llamacloud-index | `/cloud-index/getting_started/`, `/cloud-index/guides/retrieval/advanced/`, `/cloud-index/guides/parsing_transformation/` | done | retrieval-tuning.md, ingestion.md |
| llamacloud-classify | `/classify/` | done | classification-design.md |
| llamacloud-split | `/split/` | done | split-strategy.md |
| llamacloud-sheets | `/sheets/` | done | sheets-usage.md |
| liteparse | `/liteparse/`, `/liteparse/guides/library-usage/` | done | liteparse-vs-llamaparse.md |
| llamacloud-self-hosting | `/self_hosting/`, `/self_hosting/installation/` | done | deployment-checklist.md, troubleshooting.md |
| llamaindex-framework | `/python/framework/understanding/rag/`, `/python/framework/module_guides/` | done | rag-pipeline.md, component-map.md |
| llamaindex-workflows | `/python/llamaagents/workflows/`, `/python/llamaagents/workflows/deployment/` | done | workflow-patterns.md |
| llamaindex-deploy | `/python/llamaagents/llamactl/getting-started/`, `/.../llamactl/workflow-api/` | done | deployment-flow.md |
| llamaindex-cookbooks | `/llamaparse/cookbooks/`, `/python/examples/` | done | recipe-finder.md |

Update each row's status to `done` as its skill is finalized.
