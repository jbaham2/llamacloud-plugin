# Vendored / Wired Assets

This plugin **wires** one external asset rather than duplicating it. No third-party source
code is copied into the repo, so no foreign LICENSE is vendored.

## LlamaIndex Documentation MCP server (wired, not copied)

- **What:** The official hosted LlamaIndex documentation MCP server.
- **Endpoint:** `https://developers.llamaindex.ai/mcp`
- **Transport:** `http` (HTTPS)
- **Auth:** none (public, no API key required)
- **Tools exposed:** `search_docs` (BM25 lexical search), `grep_docs` (regex), `read_doc` (full page).
- **Where wired:** `.mcp.json` at plugin root, server key `docs`.
- **Tool names inside this plugin:**
  `mcp__plugin_llamacloud_docs__search_docs`,
  `mcp__plugin_llamacloud_docs__grep_docs`,
  `mcp__plugin_llamacloud_docs__read_doc`.
- **Verified:** 2026-06-17 via the docs page at `/python/shared/mcp/`.

### Why wired, not rebuilt
Operating principle #1 (vendor + complement, never duplicate): an official docs-search server
already exists, so the plugin wires it for live fact lookups instead of building a competing one.

## Live-fetch fallback (no MCP)
Any page on `https://developers.llamaindex.ai/...` is available as clean markdown by appending
`index.md` to the page URL (e.g. `…/llamaparse/parse/index.md`). Skills point to this for
`WebFetch` when the MCP is unavailable.

## Ownership boundary
The MCP/live docs own **stale-prone facts** (API signatures, parameter names, version numbers,
pricing, exact code). The plugin's skills own **durable judgment** (which product when, decision
tables, failure taxonomies, prod-readiness checklists, sequencing). No skill reproduces what the
MCP can answer live.
