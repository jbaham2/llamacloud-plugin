---
description: Guided LlamaCloud onboarding — account, API key, region choice, SDK install, and a first-call smoke test.
argument-hint: [optional: target product, e.g. "parse" or "extract"]
allowed-tools: ["mcp__plugin_llamacloud_docs__search_docs", "mcp__plugin_llamacloud_docs__read_doc", "Bash"]
---

Run the user through LlamaCloud onboarding. Target product (optional): **$ARGUMENTS**

Follow the onboarding sequence from the `llamacloud` skill's `references/onboarding.md`. Walk the
user through these steps, confirming each before moving on:

1. **Account + API key.** Have them create an account and an API key in the LlamaCloud console. The
   whole platform uses one credential, exposed as `${LLAMA_CLOUD_API_KEY}`. Tell them to export it in
   their shell / secrets manager — never hardcode or paste it here.

2. **Region (one-time, non-reversible).** Help them choose:
   - **NA** — newest features first; default.
   - **EU** — data stored & processed in-region; choose for GDPR / EU residency. Cannot migrate later.
   Confirm they understand the choice is permanent before they create the account in that region.

3. **Install the SDK.** Fetch the current package name and install command live with `search_docs` /
   `read_doc` (it has shifted between `llama-cloud` and `llama-cloud-services`) — do not assert it
   from memory. Give both Python and TypeScript install commands.

4. **Wire the key (and EU base URL if chosen).** Show the minimal client construction reading
   `LLAMA_CLOUD_API_KEY` from the environment, with the EU base URL set if they picked EU.

5. **Smoke test.** Run the smallest possible operation for the target product (default to Parse if
   `$ARGUMENTS` is empty) to verify the key works. If a key is present in the environment, you may
   use Bash to run a tiny verification; otherwise give them the snippet to run themselves. Never echo
   the key value.

6. **Hand off.** Once authenticated and regioned, point them to the product skill for the real build
   (`llamacloud-parse`, `llamacloud-extract`, `llamacloud-index`, …) or suggest `/llamacloud` to route.

Keep it conversational and one step at a time. Surface the common first-run failures (401 = key
unset/wrong region; NA/EU base-URL mismatch; wrong package version) only if the user hits them.
