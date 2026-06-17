# ROADMAP — llamacloud plugin build

Status legend: ☐ todo · ◐ in-progress · ☑ done

## Phase 0 — Foundation
- ☑ Verify official docs-MCP + clean-markdown fetch
- ☑ Skeleton (`.claude-plugin/`, `skills/`, `agents/`, `commands/`, `hooks/`, `meta/`, `workbench/`)
- ☑ `plugin.json`, `marketplace.json`, `.mcp.json`, `.gitignore`, `LICENSE`, `VENDORED.md`
- ☑ `meta/DECISIONS.md`, `meta/ROADMAP.md`, source tracker
- ◐ README.md

## Phase 1 — Template skill
- ◐ `llamacloud` (core router/overview + onboarding)

## Phases 2..N — Pillar skills (all authored, reviewed, fixes applied)
- ☑ `llamacloud-parse`
- ☑ `llamacloud-extract`
- ☑ `llamacloud-index`
- ☑ `llamacloud-classify`
- ☑ `llamacloud-split`
- ☑ `llamacloud-sheets`
- ☑ `liteparse`
- ☑ `llamacloud-self-hosting`
- ☑ `llamaindex-framework`
- ☑ `llamaindex-workflows`
- ☑ `llamaindex-deploy`
- ☑ `llamaindex-cookbooks`

## Phase N+1 — Tooling layer
- ☑ Agents: `extraction-schema-designer`, `rag-architect`, `self-hosting-doctor` (ro), `cookbook-scout` (ro), `workflow-builder`
- ☑ Commands: `/llamacloud`, `/llamacloud-setup`
- ☑ Hooks: decided NO hooks (see DECISIONS D8 — low-noise guardrail)

## Phase N+2 — Hardening & release
- ☑ skill-reviewer per skill (13× — all PASS, fixes applied; see DECISIONS)
- ☑ Trigger-collision audit across 13 skills + agents + commands (clean — `trigger-ownership-map.md`)
- ☑ README, LICENSE, .gitignore, VENDORED.md
- ☑ Final full-plugin plugin-validator — **SHIP** (0 critical, 0 warnings)
- Version: 0.1.0 (initial release)

## Status: COMPLETE — ship-ready (2026-06-17)
13 skills · 5 agents · 2 commands · 1 wired MCP. All reviewed, all validated.
