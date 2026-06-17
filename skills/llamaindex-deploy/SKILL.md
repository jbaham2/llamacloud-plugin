---
name: llamaindex-deploy
description: This skill should be used when the user wants to "deploy a workflow/agent", "use llamactl", "serve a LlamaIndex Workflow as an HTTP API", or set up "agent deployment / continuous deploy" on LlamaCloud. Owns shipping a Workflow/agent as a running service (install/auth/init/serve/deploy via the CLI, the HTTP API surface, CD hooks); defers authoring workflow logic to `llamaindex-workflows` and self-hosting the platform to `llamacloud-self-hosting`.
version: 0.1.0
---

# Deploy LlamaIndex Workflows & Agents (llamactl + LlamaCloud)

This skill is **"ship my workflow/agent as a running service."** It covers the `llamactl` CLI and
LlamaCloud's managed agent deployment: install, authenticate, scaffold, serve locally, then deploy
to the cloud and operate it over HTTP. It owns *configuring a Workflow **for serving*** — registering
the workflow instance so the platform can host it — and the HTTP API the deployment exposes.

It does **not** teach how to write the Workflow itself (events, steps, state, `@step`,
`Context`) — that is `llamaindex-workflows`. It does **not** cover self-hosting the LlamaCloud
platform (BYOC / Helm / K8s) — that is `llamacloud-self-hosting`. This skill teaches the **managed**
cloud deploy.

**Deploying an agent is deploying a workflow.** In LlamaIndex an agent (e.g. `AgentWorkflow` /
`FunctionAgent`) *is* a Workflow, so "deploy my agent" follows the exact same path as below: register
the agent instance and serve it. Everywhere this skill says "workflow," an agent qualifies.

## The deploy sequence (altitude)

Run this order; per-step detail and decisions live in `references/deployment-flow.md`.

1. **Install** `llamactl` (a CLI tool; install via `uv`). Prereqs: `uv`, `git`, and Node.js if a
   template ships a frontend UI.
2. **Authenticate** — browser login for interactive use, or a token / env vars for CI.
3. **Init** — scaffold a project from a template (workflow-only, or workflow + frontend UI).
4. **Serve locally** — run the dev server, exercise the workflow over HTTP, iterate.
5. **Commit & push to Git** — cloud deploys build *from a Git repo*. Code that isn't committed and
   pushed will not ship.
6. **Deploy to the cloud** — create a deployment from the repo + git ref; inject secrets by
   reference, not value.
7. **Operate** — list deployments, follow logs, and redeploy (branch-tracking or a pinned ref).

## Configure a Workflow for serving (this skill's boundary)

The Workflow's *logic* belongs to `llamaindex-workflows`. What belongs **here** is making an existing
Workflow **deployable**: exposing the workflow as an importable instance and **registering** it in
the project config so the server can find and run it. Registration maps a public workflow name to a
`module:instance` reference (illustratively under a `[tool.llamaagents.workflows]`-style table in
`pyproject.toml` — **verify the exact key live**, see "Fetching facts"). Multiple workflows can be
registered in one deployment. If the project has a frontend, a `ui` setting points the server at it.

Hand the *authoring* of that instance — its events, steps, and state — to `llamaindex-workflows`;
return here once there is a Workflow object to register.

## Key decisions

| Decision | Choose | When |
|---|---|---|
| **Auth mode** | Browser `auth login` | Local, interactive dev |
| | Token / env vars (`LLAMA_CLOUD_API_KEY`, project id) | CI, non-interactive, headless |
| **Template** | Workflow-only | Pure API backend |
| | Workflow + frontend UI | Need a served UI (requires Node.js) |
| **Where to run** | Local serve | Develop, debug, observe before shipping |
| | Cloud deployment | Stable, shareable, externally reachable service |
| **Redeploy mode** | Branch-tracking | Continuous deploy: re-resolve a branch, ship latest commit |
| | Pinned git-ref | Reproducible / rollback to an exact commit |
| **Deploy interface** | CLI (`llamactl`) | Scriptable, CI-friendly, repeatable specs |
| | LlamaCloud dashboard | One-off starter-template deploy from the UI |

For repeatable/GitOps deploys, generate a deployment spec to a YAML file and `apply` it, rather than
editing the spec interactively each time. See `references/deployment-flow.md`.

## CLI skeleton (illustrative — verify flags live)

Command *names* are stable; flags drift, so confirm them via the docs/MCP before scripting:

```bash
uv tool install -U llamactl        # install the CLI
llamactl auth login                # browser auth (or set LLAMA_CLOUD_API_KEY for CI)
llamactl init                      # scaffold from a template
llamactl serve                     # run locally; serve workflows + debugger UI
# commit & push to Git first — cloud builds from the repo, not your working dir
llamactl deployments create        # one-off cloud deploy (spec opens for review)
llamactl deployments template > deployment.yaml   # repeatable spec
llamactl deployments apply -f deployment.yaml      # GitOps-style apply
llamactl deployments logs NAME --follow            # stream logs
llamactl deployments update NAME                   # redeploy (branch-tracking CD)
```

Workflow registration lives in the project config, illustratively:

```toml
# pyproject.toml — verify the exact table/key live
[tool.llamaagents.workflows]
my-workflow = "my_package.module:workflow_instance"
```

## The HTTP API surface (the model, not the paths)

A deployed workflow is reachable over HTTP. Think in terms of the **capabilities**, not memorized
paths (paths drift — fetch them live):

- **Run** a workflow — POST a start event / input and get a run handle.
- **Stream events** published by an in-progress run, then retrieve the final result.
- **Send events into a running workflow** — the hook for **human-in-the-loop (HITL)**: a step waits
  on an external event, the client posts it in.
- **Introspect** registered workflows.
- **OpenAPI docs** are served per deployment (a `/docs`-style path) — the authoritative, live
  description of the actual endpoints for *that* deployment. Read it instead of hardcoding routes.

Illustrative shape (verify live): `POST /deployments/{name}/workflows/{workflow}/run` with a JSON
body, plus a `Bearer` token for cloud deployments. During local dev a **debugger UI** is served for
inspecting runs. Get exact endpoints, the start-event JSON shape, and the local port from the
deployment's own OpenAPI page or the workflow-api docs below.

## Secrets, state, and persistence

- **Secrets:** never hardcode. Deployment specs reference local env vars (e.g. an
  `OPENAI_API_KEY: ${OPENAI_API_KEY}` style entry) so values are injected at deploy time. Keep keys
  out of the repo and out of logs.
- **State / persistence:** the deploy docs are thin on whether a served workflow's `Context` /
  session state persists across requests or restarts. **Do not assume durability.** If the workflow
  is stateful (HITL, multi-turn), treat persistence as an open question to confirm via the docs/MCP
  before relying on it, and design for restarts.

## Common failure: Git is the source of truth

The canonical deploy failure: **cloud deployments build from a Git repository.** If local code is
uncommitted or unpushed, the deploy ships the *old* commit — symptoms look like "my change didn't
take effect." **Private repos require repository access to be configured first**, or the build can't
clone. The failure taxonomy (auth, registration, secret-injection, git/ref, UI/Node issues) is in
`references/deployment-flow.md`.

## Cross-links

- Authoring the Workflow (events / steps / state / `Context`) → **`llamaindex-workflows`**.
- Self-hosting the LlamaCloud platform (BYOC / Helm / K8s) → **`llamacloud-self-hosting`**.
- Account, API key, region selection, product routing → **`llamacloud`**.

## Fetching facts (do not hardcode)

Exact CLI flags, the workflow-registration config key, endpoint paths, the local dev port, package
names/versions, and base URLs **drift** — fetch them live, never from memory:

- **Docs MCP** (wired, no auth): `mcp__plugin_llamacloud_docs__search_docs` (concepts),
  `mcp__plugin_llamacloud_docs__read_doc` (a known page), `mcp__plugin_llamacloud_docs__grep_docs`
  (exact symbol / config key / endpoint).
- **Fallback:** append `index.md` to any `https://developers.llamaindex.ai/...` page URL and
  `WebFetch` it.
- **For exact run/stream/HITL endpoints of a live deployment:** read that deployment's own
  **OpenAPI docs page** — it is always current for that build.

Anchor pages:
- `/python/llamaagents/llamactl/getting-started/` — install, auth, init, serve, deploy, operate.
- `/python/llamaagents/llamactl/workflow-api/` — the HTTP API surface and workflow registration.

## Additional resources

- **`references/deployment-flow.md`** — the init→serve→commit→deploy→operate sequence with what to
  decide at each step, the repeatable-spec (YAML apply) pattern, serving/persistence considerations,
  and the deployment failure taxonomy.
