# DECISIONS — llamacloud plugin

Append-only record of architecture decisions and tool verdicts.

## D1 — Single plugin, skills-per-domain (2026-06-17)
13 knowledge **skills** (auto-trigger), 5 delegated **agents** (go-do-X), 2 **commands** (launchers),
1 wired **MCP** (live docs). Mirrors the Langfuse plugin shape. Source: `plugin-plan.md` §5 (resolved).

## D2 — MCP: wire official, don't build (2026-06-17)
Verified live: `https://developers.llamaindex.ai/mcp`, transport `http`, **no auth**, tools
`search_docs`/`grep_docs`/`read_doc`. Wired in `.mcp.json` under server key `docs`. The API key
(`${LLAMA_CLOUD_API_KEY}`) is **not** needed by the docs MCP — only by user runtime code. See `VENDORED.md`.

## D3 — Distill judgment, fetch facts (2026-06-17)
`references/*.md` hold durable judgment only (decision tables, failure taxonomies, checklists,
sequencing). API signatures / params / versions / pricing are fetched live via the MCP or
`index.md`-append. Each SKILL.md points to the live source for facts.

## D4 — Version posture (authoritative, encoded in every skill)
Parse **v2** + Extract **v2** = primary GA surface. Their `/v1/` trees = legacy / migration-reference only.
Cloud Index **v2** = beta, deferred (skill teaches stable Cloud Index; flags v2 as beta). LiteParse =
local/OSS, no API key. Never present legacy/beta as the default path.

## D5 — Session scope: full build to definition-of-done (2026-06-17)
User chose "Everything": skeleton → 13 skills → 5 agents → 2 commands → hooks → validate → release.

## D6 — Python + TypeScript both inline (2026-06-17)
User chose dual-SDK inline examples (`llama-cloud-services` Python + TS). Tension with "fetch facts
live" is mitigated by keeping inline snippets minimal/illustrative and pointing to the MCP for full
signatures. Recorded as an accepted trade-off.

## D7 — Plugin root = working dir (2026-06-17)
Plugin root is the repository root (the `llamacloud-plugin` checkout). Planning `.md` files +
`workbench/` are gitignored so only plugin dirs ship.

## D8 — No hooks shipped (2026-06-17)
Considered a non-blocking PostToolUse "nudge" when v1 Parse/Extract SDK patterns appear in user
writes. Rejected: detecting `/v1/`-style usage reliably is hard, and a write-scanning hook produces
false positives (any string containing "v1") — violating the low-noise guardrail. The v2-primary /
v1-legacy posture is already encoded in every skill's body, which is higher-signal and zero-noise.
The empty `hooks/` dir was removed (no empty scaffolding).

## Tool verdicts
- skill-development, mcp-integration, agent-development skills: consulted (Phase 0 / N+1).
- plugin-validator agent (skeleton, Phase 0): **PASS**, 0 issues.
- skill-reviewer agent ×13 (Phase N+2): all **PASS**. Fixes applied:
  - `llamacloud-sheets`: removed hardcoded throttle/size numbers (`1 req/s`, `100 cols/10k rows`) — MAJOR.
  - `llamacloud-extract`: removed bare `llama-cloud-services` heading assertion → "confirm via MCP" — MAJOR.
  - `llamacloud` (core): version posture kept authoritative per user directive, added "GA/beta labels
    drift — verify live" caveat to resolve the self-contradiction the reviewer flagged; fixed one
    second-person line in onboarding.md.
  - `llamacloud-self-hosting`: `Temporal` → `workflow-engine` in deployment-checklist.md (de-pinned engine name).
  - `llamaindex-cookbooks`: prose handoff → `` `llamacloud-index` `` (machine-followable).
  - Accepted as-is (reviewer-rated minor, and consistent with D6 inline-examples choice): caveated
    literal enums/tier names/param names inside illustrative snippets across parse/extract/index/etc.
- plugin-validator agent (final, full plugin): **SHIP** — 0 critical, 0 warnings. Confirmed: all JSON
  valid, name/dir match on all 13 skills, all 19 references resolve, all `mcp__plugin_llamacloud_docs__*`
  tool names match the live server (which also exposes `list_docs`), read-only agents lack `Write`,
  no secrets. Ship-ready.

## D9 — Reviewer "MCP not in session" note is a non-issue (2026-06-17)
Several skill-reviewers noted the `mcp__plugin_llamacloud_docs__*` tools weren't present in their
session. Expected: subagents don't have the plugin installed. The endpoint was verified live in
Phase 0 and `.mcp.json` is structurally correct (matches the registered langfuse plugin's form).
The `index.md`-append + WebFetch path is the always-available fallback.
