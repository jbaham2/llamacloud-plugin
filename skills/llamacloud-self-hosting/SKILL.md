---
name: llamacloud-self-hosting
description: This skill should be used when the user wants to "self-host LlamaCloud", do a "BYOC deployment", "Helm/Kubernetes install LlamaParse/LlamaCloud", work out "self-hosting installation/hardware requirements", or "tune/monitor a self-hosted LlamaCloud". Owns operating the MANAGED LlamaCloud platform inside your own cloud (Helm/K8s, external services, LLM/provider wiring, sizing, monitoring); defers deploying your own agents/workflows to llamaindex-deploy and product usage to the product skills.
version: 0.1.0
---

# Self-hosting LlamaCloud (BYOC)

Run the full managed LlamaCloud platform — LlamaParse, LlamaExtract, managed
indexes, Web UI, REST API — inside your own Kubernetes cluster via a Helm chart.
This is an **operations / deployment** skill, not a product-usage skill.

## When this applies (and when it does not)

Use this skill to plan, install, size, secure, tune, or troubleshoot a BYOC
LlamaCloud cluster. It is the knowledge base behind the read-only
`self-hosting-doctor` agent.

Do NOT use it for:
- Deploying YOUR OWN LlamaIndex agents/workflows (`llamactl`) → `llamaindex-deploy`.
- Using the parsing/extraction products themselves → `llamacloud-extract`,
  `liteparse`, and the parse product skill.
- Account creation, API keys, plan/product choice on the SaaS platform → core `llamacloud`.

If the user is on managed SaaS and just wants to call the API, they do not need
BYOC — redirect to core `llamacloud`.

## What BYOC gives you (decide if it's warranted)

The selling point is **data residency and isolation**: documents, embeddings, and
processed outputs never leave your VPC. You also get enterprise auth (OIDC against
your IdP) and the ability to pin a single LLM provider. The cost is that you own a
non-trivial Kubernetes deployment plus its external datastores.

Pick BYOC only when a real constraint forces it — regulated data, VPC-only
networking, contractual residency. Otherwise managed SaaS is far less operational
burden. BYOC requires an Enterprise plan and a **license key** (obtained through
LlamaIndex sales/support — confirm the current contact via the docs, do not
hard-code an address).

## Security model (the load-bearing concept)

- **Control stays in your `values.yaml`.** Your cluster runs the whole platform;
  there is no managed control plane reaching into it at runtime. The license key
  gates entitlement, not data flow.
- **Data plane is your VPC.** Files land in YOUR object store (S3/Blob/GCS),
  metadata in YOUR Postgres/Mongo, queues/state in YOUR RabbitMQ/Redis/Temporal.
- **Egress is the LLM call.** The one boundary crossing is the outbound request to
  the LLM provider. For strict isolation, terminate that inside your perimeter too
  — Azure OpenAI, AWS Bedrock, or Vertex AI in your own account/region.
- **Auth posture by environment.** OIDC for production; basic auth is acceptable
  only for staging/throwaway clusters.

See `references/deployment-checklist.md` for the network/security model as a
pre-flight checklist.

## Architecture and external services

LlamaCloud is a set of services on Kubernetes that depend on **external
datastores you provision** (managed cloud services strongly preferred over
in-cluster pods for anything stateful). The dependency set spans a relational
store, a document store, a workflow/orchestration engine, a message broker, and a
cache. The orchestration engine can optionally run as a subchart for evaluation,
but production should use a managed/standalone instance.

Treat these as the four BYOC infrastructure decisions, made before `helm install`:
1. **Stateful backends** — managed (RDS/Cloud SQL, Atlas, etc.) vs in-cluster.
2. **Object storage** — which provider bucket and how its IAM is scoped.
3. **LLM provider/endpoint** — public API vs in-cloud (Bedrock/Vertex/Azure).
4. **Auth + ingress** — OIDC, ingress controller, TLS, public vs private.

Do not assert exact service names/versions from memory — they shift. Confirm the
current required-services list, minimum K8s/Helm versions, and architecture
against the live installation doc (below) before committing a design.

## Hardware and sizing

LlamaParse is **CPU- and memory-intensive**; size for the parse workers, not the
API. Key durable facts to verify live (they change): the platform is **x86/amd64
only** (no arm64), needs a modern Linux, and has a non-trivial vCPU/memory floor
for a single-node trial. Production capacity is driven by parse concurrency and
document size/complexity, so plan to scale parse workers horizontally (HPA, or
KEDA against queue depth) rather than vertically. Sizing guidance and the
scaling-signal decisions live in `references/deployment-checklist.md`.

## Installation sequence (illustrative)

Read the live installation doc for exact commands, chart name, and the current
`values.yaml` template before running anything. The shape is:

```bash
# 1. add the chart repo (confirm repo URL + chart name in the live doc)
helm repo add llamaindex https://run-llama.github.io/helm-charts
helm repo update

# 2. namespace
kubectl create namespace llamacloud

# 3. install with your values.yaml (license key, datastore creds,
#    LLM creds, object store, auth/OIDC, ingress all live here)
helm install llamacloud llamaindex/llamacloud \
  -f values.yaml --namespace llamacloud

# 4. validate
kubectl get pods -n llamacloud
kubectl port-forward -n llamacloud svc/<web-svc> 3000:3000
```

`values.yaml` is the whole game — license key, every external-service credential,
LLM keys, auth mode, storage backend, ingress, and autoscaling all live there.
Build it from the live template; keep secrets in a secret manager / sealed
secrets, never in the chart values committed to git. For the field-by-field exact
schema, fetch the installation doc or query the MCP — do not invent field names.

## LLM integration and provider support

The platform calls an LLM provider for parsing/extraction. Three durable
decisions:
- **Which provider** — OpenAI and Azure OpenAI are the smoothest path; Anthropic,
  Bedrock, and Vertex AI are also supported. Confirm the current support matrix in
  the live doc rather than assuming a model is available.
- **Public API vs in-cloud endpoint** — the public API is simplest; an in-account
  endpoint (Azure OpenAI / Bedrock / Vertex) keeps the LLM call inside your
  perimeter and is the right choice when residency is the reason you chose BYOC.
- **Single-provider pinning** — organizations can restrict to one provider/model;
  do this deliberately and store the choice in `values.yaml`.

Provider rate limits and quota are the most common runtime failure source — they
surface as intermittent parse/extract failures (429s), not as cluster errors.

## Tuning and monitoring (overview)

- **Throughput** — scale parse workers on **queue depth/age**, not CPU alone; a
  worker blocked on a slow LLM call looks idle on CPU. KEDA against the broker
  queue is the natural autoscaling signal; HPA on CPU is a weaker fallback.
- **Headroom** — leave broker and DB capacity for the retry storm that follows any
  outage; the backlog drains all at once.
- **Observability** — the doc set is thin on built-in monitoring, so wire your own.
  Scrape pod metrics, watch broker queue depth and workflow-engine health, and
  alert on object-store and DB connection errors (the failures that present as
  vague app-level 5xx). Treat the external services as first-class monitoring
  targets, not just the pods.

When something breaks, work the failure taxonomy in
`references/troubleshooting.md` (pod → datastore → queue/orchestration →
LLM-integration symptom layers). That file is written to be diagnostic and
actionable for both humans and the `self-hosting-doctor` agent.

## Version posture

This skill is product-version-agnostic at the platform level: a BYOC cluster
serves whatever Parse/Extract/Index versions the chart ships. Within the products,
**Parse v2 + Extract v2 are GA defaults**; `/v1/` is legacy/migration-reference;
**Cloud Index v2, Classify, Split, Sheets are beta** — set expectations
accordingly and defer product specifics to the product skills.

## Fetching facts (do not reproduce stale-prone values)

Keep exact chart names, `values.yaml` field names, required-service versions,
minimum K8s/Helm versions, vCPU/memory floors, and the license-key contact OUT of
memory — fetch them live:

- `mcp__plugin_llamacloud_docs__search_docs` — concept/BM25 search.
- `mcp__plugin_llamacloud_docs__read_doc` — read a known doc page.
- `mcp__plugin_llamacloud_docs__grep_docs` — exact symbol/field/regex search.
- Fallback: append `index.md` to any `https://developers.llamaindex.ai/...` page
  URL and WebFetch it.

Anchor docs for this pillar:
- `https://developers.llamaindex.ai/llamaparse/self_hosting/` (overview, security,
  providers).
- `https://developers.llamaindex.ai/llamaparse/self_hosting/installation/`
  (prerequisites, external services, hardware, Helm steps, `values.yaml`).

Always reconcile any command, version, or field below against these before acting.
