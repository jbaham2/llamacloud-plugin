# Deployment flow: decide at each step, and what breaks

Durable judgment for shipping a LlamaIndex Workflow/agent with `llamactl` to LlamaCloud. Exact flags,
config keys, ports, and endpoints drift — confirm them live (see the parent SKILL's "Fetching facts").
This file holds the *sequence, the decisions, and the failure modes*, not signatures.

## The sequence, with the decision at each step

### 1. Install
`llamactl` is a standalone CLI installed via `uv`. Prerequisites: `uv`, `git`, and **Node.js only if**
a chosen template ships a frontend UI. On Windows, Developer Mode may be required.
**Decide:** nothing yet — but install Node up front if a UI template is likely, to avoid a re-run.

### 2. Authenticate
Two modes:
- **Browser login** — interactive, stores a local credential profile. Best for a workstation.
- **Token / environment variables** (`LLAMA_CLOUD_API_KEY`, project id) — non-interactive. **Required
  for CI**, containers, and any headless runner where a browser can't open.

**Decide:** interactive vs CI. If the same machine both develops and runs CI, prefer env-var auth in
the pipeline and browser auth locally — don't bake a personal profile into CI.

### 3. Init (scaffold)
`init` opens a template picker and generates the project. Templates are either **workflow-only**
(pure Python API backend) or **workflow + frontend UI** (Python app plus a UI the server proxies).

**Decide:** UI or not.
- Workflow-only → simplest, no Node dependency, API-only consumers.
- With UI → you get a served frontend, but you now need Node.js locally and a `ui` directory setting
  pointing the dev server at the frontend.

### 4. Configure the workflow for serving
Make the Workflow **deployable**: expose it as an importable instance and **register** it in the
project config (a `module:instance` mapping under a `pyproject.toml` table — verify the exact key).
Register multiple workflows in one deployment if needed.

**Boundary:** authoring the instance (its events/steps/state) is `llamaindex-workflows`. This step is
only the wiring that makes an *existing* Workflow object servable.

**Decide:** one deployment with several workflows, or split deployments. Group workflows that share a
release cadence, secrets, and scaling profile; split when they need independent deploys.

### 5. Serve locally
The dev server installs deps, reads the workflow config, serves the workflows over HTTP, and (if the
app has a UI) proxies the frontend dev server. A **debugger UI** is available for observing runs.

**Decide:** verify each workflow runs, streams events, and (for HITL) accepts injected events here,
*before* committing. Local serve is the cheap iteration loop; the cloud build is slow and git-gated.

### 6. Commit & push to Git
**Cloud deploys build from a Git repository.** This is the single most important operational fact:
the build clones a repo at a ref — it does **not** upload your working directory.

**Decide:** which branch is the deploy source. Commit *and push* before deploying.

### 7. Create the cloud deployment
Two interfaces:
- **CLI** — create a deployment; the spec (repo URL, git ref, secrets) opens for review. For
  repeatable/GitOps flow, **generate the spec to a YAML file once, then `apply -f` it** — this is the
  preferred path for CI and for reproducibility, versus editing the spec interactively every time.
- **Dashboard** — deploy a starter template directly from the LlamaCloud UI. Good for a one-off; not
  scriptable.

**Secrets:** the spec references local env vars (e.g. `OPENAI_API_KEY: ${OPENAI_API_KEY}`) so values
are injected at deploy time, never committed. Keep secret values out of the repo and the spec file.

**Decide:** CLI+YAML (repeatable, CI) vs dashboard (one-off). Choose YAML-apply for anything you'll
redeploy.

### 8. Operate
- **List** deployments to see status.
- **Follow logs** (with a `--follow`-style flag) to watch a release or debug a run.
- **Update / redeploy:**
  - **Branch-tracking** — re-resolve the tracked branch and ship the latest commit. This is the
    **continuous-deploy** posture: push to the branch → run update → new release.
  - **Pinned git-ref** — deploy an exact commit/tag. Use for reproducibility and rollback.

**Decide:** CD vs pinned. Track a branch for fast iteration; pin refs for production stability and
auditable rollbacks. A common pattern is branch-tracking on staging, pinned refs on prod.

## Serving & persistence considerations

- **Statelessness assumption:** the deploy docs do not guarantee that a workflow's `Context` /
  session state survives across requests or restarts. **Do not assume durability.** For stateful or
  HITL workflows, confirm the persistence model in the docs/MCP before relying on it, and assume a
  restart can drop in-memory run state.
- **HITL needs a reachable client:** the "send event into a running workflow" capability only works
  if a client holds the run handle and can post the awaited event. Design the caller accordingly.
- **OpenAPI is the source of truth for routes:** each deployment serves its own OpenAPI docs page.
  Read it for exact endpoints and request shapes instead of hardcoding paths that may change.

## Failure taxonomy

| Symptom | Likely cause | Fix |
|---|---|---|
| "My change didn't take effect" | Code uncommitted or unpushed; build used an old commit | Commit **and push**; redeploy from the right ref |
| Build can't clone the repo | Private repo without configured access | Configure repository access before deploying |
| Deploy/CLI fails in CI | Browser-only auth in a headless runner | Use token / env-var auth (`LLAMA_CLOUD_API_KEY`, project id) |
| Workflow 404 / not found at runtime | Workflow not registered, or wrong `module:instance` ref | Fix the registration mapping; verify the import path resolves |
| Missing/invalid secret at runtime | Secret not referenced in the spec, or env var unset where the deploy runs | Add the `${VAR}` reference; ensure the var is set in that environment |
| UI template fails to build/serve | Node.js missing, or `ui` directory misconfigured | Install Node; point the `ui` setting at the frontend |
| Endpoint shape mismatch (run/stream) | Hardcoded a stale path or start-event shape | Read the deployment's OpenAPI page; use the current shape |
