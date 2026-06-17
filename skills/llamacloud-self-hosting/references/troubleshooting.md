# Self-hosted LlamaCloud troubleshooting taxonomy

Diagnostic, read-only failure taxonomy for a BYOC LlamaCloud cluster. Backs the
`self-hosting-doctor` agent: each layer lists **symptom → likely cause → what to
inspect (read-only) → fix direction**. Work the layers top-down — a parse failure
is far more often a downstream datastore/LLM problem than a parse bug.

Inspection is read-only (`kubectl get/describe/logs`, provider consoles). Never
mutate cluster state while diagnosing. Confirm exact resource/field names against
the live install doc.

## Layer 1 — Pods / cluster

| Symptom | Likely cause | Inspect | Fix direction |
|---|---|---|---|
| Pod `Pending` | insufficient CPU/mem; no schedulable node | `kubectl describe pod`, `kubectl get nodes`, events | add capacity / right-size node group |
| Pod `Pending` on amd64 image | scheduled to arm64 node | node arch labels | x86/amd64 nodes only — no arm64 |
| `CrashLoopBackOff` at startup | bad/missing config or secret | `kubectl logs --previous`, `describe` | fix `values.yaml` / missing secret |
| `ImagePullBackOff` | registry auth / license entitlement | describe events | check pull secret / license validity |
| `OOMKilled` (parse worker) | parse memory floor too low | `describe` (last state), metrics | raise worker mem requests/limits |
| Readiness probe failing | app up but can't reach a backend | logs | jump to the datastore layer |

## Layer 2 — Datastores (relational / document / cache / object store)

App pods healthy but requests error or hang → almost always a backend.

| Symptom | Likely cause | Inspect | Fix direction |
|---|---|---|---|
| Boot fails, DB connection error | wrong host/creds; SG/firewall blocks | app logs; test reachability from cluster | fix conn string; open SG to cluster CIDR |
| Intermittent 5xx under load | DB connection pool / max-connections exhausted | DB metrics; app pool errors | raise DB instance / pool; separate app vs workflow DB |
| Parse "succeeds" but no output file | object-store write denied | app logs (403/AccessDenied) | widen IAM to the bucket/prefix; check region/endpoint |
| Slow everything, cache errors | cache unreachable / undersized | cache metrics; app logs | restore/scale cache |
| Workflow engine + app both flaky | shared DB instance overloaded | DB CPU/connections | separate the failure domains |

## Layer 3 — Queue / orchestration (broker + workflow engine)

Jobs accepted but never finish → the async tier.

| Symptom | Likely cause | Inspect | Fix direction |
|---|---|---|---|
| Jobs queue forever, never run | no/too-few parse workers consuming | worker pod count; broker consumer count | scale workers; check worker crashloop (Layer 1) |
| Queue depth climbs unbounded | ingestion > parse throughput | broker queue depth/age | autoscale workers on queue depth (KEDA) |
| Workflows stuck / timing out | workflow engine unhealthy or its DB down | engine UI/logs; its DB (Layer 2) | restore engine; check its backing DB |
| Backlog explodes after an outage | retry storm draining at once | broker depth | ensure broker/DB headroom; throttle ingestion |
| Subchart workflow engine flaky | in-cluster engine not prod-grade | engine pod | move to managed/standalone for prod |

## Layer 4 — LLM integration

Parse/extract jobs run but fail or degrade in quality at the model call.

| Symptom | Likely cause | Inspect | Fix direction |
|---|---|---|---|
| Parse/extract fails with 401/403 | bad/expired LLM API key | worker logs (provider error) | rotate key in secret; reload |
| Intermittent failures, 429s | provider rate limit / quota | worker logs; provider console quotas | raise quota; back off; cap concurrency |
| Calls time out / hang | egress blocked to provider; wrong endpoint | egress policy; endpoint in `values.yaml` | open egress or use in-cloud endpoint |
| Works in trial, fails in locked-down prod | no egress to public LLM API | network policy | switch to Bedrock/Vertex/Azure in-VPC |
| Output quality regressed | wrong/changed model pin | configured model vs intended | re-pin provider/model per residency decision |
| Residency concern flagged | using public LLM API | configured endpoint | move to in-account provider endpoint |

## Layer 5 — Auth / ingress / networking

Platform healthy internally but users can't reach or log in.

| Symptom | Likely cause | Inspect | Fix direction |
|---|---|---|---|
| `port-forward` works, ingress doesn't | ingress/LB/DNS misconfig | ingress object, LB, DNS | fix ingress class / DNS / cert |
| TLS errors | cert missing/expired/mismatched | cert + ingress TLS config | issue/renew cert |
| OIDC login loops or 401 | redirect URI / client / issuer mismatch | OIDC config vs IdP app | align redirect URI, client id/secret, issuer |
| Public exposure unexpected | ingress public by default | ingress annotations/scope | make internal; expose only deliberately |

## Cross-cutting diagnostic order (for the doctor agent)

1. `kubectl get pods -n <ns>` — any pod not Running/Ready? → Layer 1.
2. All pods Ready but errors? Check logs for which backend → Layer 2.
3. Jobs accepted, never complete? Workers + queue + workflow engine → Layer 3.
4. Jobs run, model step fails/slow/wrong? → Layer 4.
5. Internally healthy, externally unreachable? → Layer 5.

Report findings as observations + recommended (non-destructive) next step; do not
apply changes. Reconcile resource names and the required-services list with the
live installation doc before asserting specifics.
