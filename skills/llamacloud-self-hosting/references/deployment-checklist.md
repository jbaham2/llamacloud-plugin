# BYOC LlamaCloud deployment checklist

Durable judgment for planning a self-hosted install. Fetch exact versions, chart
names, and `values.yaml` fields from the live installation doc — this file is the
sequencing, decisions, and gotchas.

## Phase 0 — Decide whether to self-host

Self-host only if a hard constraint demands it:
- Regulated data that cannot leave a VPC / region.
- Contractual data-residency or no-third-party-egress requirements.
- Network policy that forbids calling managed SaaS.

If none apply, managed SaaS is the right answer — BYOC adds a Kubernetes platform
and 4+ stateful backends to own. Confirm you have: an Enterprise plan, a license
key (via LlamaIndex sales/support — get current contact from the docs), and a team
that operates Kubernetes.

## Phase 1 — External-services decisions (before `helm install`)

Provision these and decide managed-vs-in-cluster for each. Default: **managed
cloud services for everything stateful**; in-cluster pods only for throwaway
trials.

| Backend role | Managed (prod default) | In-cluster (trial only) | Notes |
|---|---|---|---|
| Relational store | RDS / Cloud SQL / Azure DB | chart pod | also backs the workflow engine |
| Document store | managed equivalent | chart pod | |
| Workflow/orchestration | standalone/managed | subchart toggle | subchart fine for eval, not prod |
| Message broker | managed | chart pod | parse-queue backbone; drives autoscaling signal |
| Cache | managed | chart pod | |
| Object storage | S3 / Azure Blob / GCS | n/a | scope IAM to one bucket/prefix |

Get the authoritative service list, names, and versions from the live install doc
— do not rely on the table labels above for exact product names.

Gotchas:
- Backing the workflow engine and the app on the **same DB instance** is convenient
  for trials but couples failure domains — separate in prod.
- Object-store IAM should be **least-privilege to one bucket/prefix**, not account-wide.
- Put every credential in a secret manager; never commit them in `values.yaml`.

## Phase 2 — Sizing

- Size for **LlamaParse workers** (CPU+memory heavy), not the API tier.
- Verify the current single-node floor (vCPU/memory), arch (x86/amd64 only — no
  arm64), and OS minimum against the live doc before picking instance types.
- Production capacity scales with **parse concurrency × document complexity**.
  Plan to scale parse workers **horizontally**.
- Scaling signal: **queue depth / age** (KEDA against the broker) beats raw CPU,
  because a worker can be saturated on a slow LLM call while CPU looks idle.
- Headroom: leave the broker and DBs with capacity for retry storms after an
  outage — backlog drains all at once.

## Phase 3 — Security & network model

- **Data plane = your VPC.** Files → your object store; metadata → your DBs;
  queue/state → your broker/cache/workflow engine.
- **Only egress = the LLM call.** For strict isolation, keep it in-perimeter:
  Azure OpenAI / Bedrock / Vertex in your own account & region. Public OpenAI/
  Anthropic/Google APIs are simplest but cross the boundary.
- **Auth:** OIDC against your IdP for production; basic auth only for staging.
- **Ingress:** TLS terminated; default to **private/internal** ingress, expose
  publicly only with an explicit reason.
- **License key** gates entitlement only — it is not a data path. Store it as a secret.

## Phase 4 — Rollout phases

1. **Trial** — single node, in-cluster backends, basic auth, public LLM API.
   Goal: prove the chart installs and a document parses end-to-end.
2. **Staging** — managed backends, OIDC, private ingress, real instance sizing.
   Goal: validate auth, networking, and the residency story.
3. **Production** — separated DB failure domains, autoscaling (HPA/KEDA on queue),
   secret manager, monitoring/alerting wired, backups on stateful stores.
4. **Steady state** — load-test parse concurrency, set HPA/KEDA thresholds from
   observed queue behavior, document upgrade/rollback runbook.

## Pre-go-live gate

- [ ] License key + all creds in a secret manager, none in committed values.
- [ ] Managed (or HA) backends; app and workflow-engine DBs not single-pointed.
- [ ] Object-store IAM least-privilege to one bucket/prefix.
- [ ] OIDC working; ingress TLS; ingress private unless justified.
- [ ] LLM endpoint matches the residency decision; rate limits understood.
- [ ] Parse workers autoscale on queue depth; broker/DB headroom for retry storms.
- [ ] Metrics scraped; alerts on DB/object-store/broker/workflow-engine errors.
- [ ] Backups + a tested restore for each stateful store.
- [ ] Upgrade/rollback runbook (chart version pinned, `helm rollback` rehearsed).
