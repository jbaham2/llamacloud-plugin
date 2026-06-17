# Trigger Ownership & Collision Audit

Audited the `description:` trigger phrases of all 13 skills + 5 agents + 2 commands for unintended
collisions. **Verdict: no harmful collisions.** Soft overlaps exist only on generic verbs and are
disambiguated by required qualifier words plus explicit defer-clauses in each description.

## Skill ownership (one owner per intent)

| Intent / utterance | Owner | Disambiguator |
|---|---|---|
| getting started, which product, API key, regions, pricing | `llamacloud` | setup/routing scope only |
| parse a PDF, LlamaParse tiers, configure parse output | `llamacloud-parse` | cloud parsing verbs |
| parse locally / no key / offline / LiteParse | `liteparse` | "locally", "no key", "LiteParse" |
| extract structured data, schema, typed fields from PDFs | `llamacloud-extract` | "schema"/"typed fields"/document |
| extract from spreadsheets / Excel / CSV | `llamacloud-sheets` | "spreadsheet"/"Excel"/"CSV" |
| classify / route by type / categorize | `llamacloud-classify` | "type"/"route"/"categorize" |
| split / segment a concatenated file | `llamacloud-split` | "split"/"segment"/"concatenated" |
| managed RAG, Cloud Index, hybrid/rerank on LlamaCloud | `llamacloud-index` | "Cloud Index"/"managed" |
| RAG in code, VectorStoreIndex, OSS framework | `llamaindex-framework` | "OSS"/"VectorStoreIndex"/"in code" |
| build a Workflow, event-driven steps, HITL/streaming | `llamaindex-workflows` | "Workflow"/"event-driven" |
| deploy/serve a workflow, llamactl, HTTP API | `llamaindex-deploy` | "deploy"/"serve"/"llamactl" |
| self-host, BYOC, Helm/K8s install the platform | `llamacloud-self-hosting` | "self-host"/"BYOC"/"Helm" |
| find an example/cookbook/recipe for X | `llamaindex-cookbooks` | "example"/"cookbook"/"recipe" |

## Soft overlaps reviewed (all acceptable)

1. **`llamacloud` (core) vs `llamacloud-parse` on "LlamaParse".** Core triggers on *getting
   started / set up / which product*; parse on *parse a PDF / tiers / config*. If core fires on a
   bare "use LlamaParse", it routes onward — core explicitly defers product depth. No harm.
2. **`llamacloud-index` vs `llamaindex-framework` on "RAG".** Both qualify the verb: index requires
   *Cloud Index / managed / on LlamaCloud*; framework requires *OSS / VectorStoreIndex / in code*.
   A bare "build RAG" is intentionally routed by core `llamacloud` and the `rag-architect` agent.
3. **`llamaindex-workflows` vs `llamaindex-deploy` on "workflow".** build/author vs deploy/serve;
   cross-defer clauses present.
4. **`llamacloud-self-hosting` vs `llamaindex-deploy` on "deploy".** Hosting the *platform* (Helm/K8s)
   vs deploying *your own agent* (llamactl); both name the boundary explicitly.
5. **`llamacloud-extract` vs `llamacloud-sheets` on "extract".** document/PDF vs spreadsheet/Excel/CSV;
   cross-defer present.

## Discriminating-utterance checks (advisor's test)
- "how do I use LlamaParse" → `llamacloud` (routes) or `llamacloud-parse` — both fine; core defers. ✓
- "extract data from this PDF" → `llamacloud-extract` (not core). ✓
- "extract from this spreadsheet" → `llamacloud-sheets` (not extract). ✓
- "build a Cloud Index" → `llamacloud-index` (not framework). ✓
- "build a RAG pipeline in code" → `llamaindex-framework` (not index). ✓
- "deploy my workflow" → `llamaindex-deploy` (not workflows/self-hosting). ✓

## Agents vs skills
Agents trigger on *"design/scaffold/diagnose/find for me"* (delegated action), skills on *knowledge*.
`extraction-schema-designer` ↔ `llamacloud-extract`, `rag-architect` ↔ `index`/`framework`,
`self-hosting-doctor` ↔ `self-hosting`, `cookbook-scout` ↔ `cookbooks`, `workflow-builder` ↔
`workflows` — each agent does, the paired skill teaches. No collision.
