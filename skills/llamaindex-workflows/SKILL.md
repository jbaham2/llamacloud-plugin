---
name: llamaindex-workflows
description: This skill should be used when the user wants to "build a LlamaIndex Workflow", design "event-driven agent steps", handle "workflow state / HITL / streaming", or "orchestrate agentic steps with Workflows". Owns the event-driven workflow model (events, steps, Context/state, branching, looping, human-in-the-loop, streaming, durability) and the judgment of when a Workflow is the right abstraction; defers serving a workflow as an API to `llamaindex-deploy` and plain RAG component wiring to `llamaindex-framework`.
version: 0.1.0
---

# LlamaIndex Workflows — Event-Driven Orchestration

Workflows are LlamaIndex's **event-driven** abstraction for orchestrating multi-step, agentic
logic. Instead of wiring a DAG of nodes, define `@step` methods (Python) / handlers (TypeScript)
that **consume an event and emit one or more events**. The framework infers the graph from those
input/output event types and runs steps as their inputs arrive — so branching, looping, fan-out,
and joins are expressed in ordinary async Python/TS, not in graph edges.

This skill holds the **mental model and design judgment**. For exact class/method signatures, fetch
live (see "Fetching facts"). For *serving* a finished workflow over HTTP, defer to
`llamaindex-deploy`. For assembling plain RAG components (loaders, indexes, query engines, retrievers)
without orchestration, defer to `llamaindex-framework`.

## When a Workflow is the right abstraction

Reach for a Workflow only when the control flow itself is the problem. Decision shortcuts:

- **Single query → answer, linear** (retrieve → synthesize) → use a **query engine / chain** from
  `llamaindex-framework`. A Workflow adds ceremony with no payoff.
- **Branching on intermediate results, loops/retries, fan-out + join, or multiple cooperating
  steps** → **Workflow**. This is the sweet spot.
- **Needs to pause for a human, then resume** (approvals, clarifying questions) → **Workflow** with
  human-in-the-loop. Nothing else in the stack pauses cleanly.
- **Long-running / must survive a crash or restart** → **Workflow** with durability/checkpointing.
- **Streaming intermediate progress to a UI** → **Workflow** emits events to a stream as it runs.
- **Just calling one prebuilt agent (e.g. `FunctionAgent`)** → use the agent directly; reach for a
  custom Workflow once you need to orchestrate *around* or *between* agents.

See `references/workflow-patterns.md` for the full when-to-use table and the build sequence.

**Workflow vs. multi-agent orchestration.** When several agents must cooperate, the choice is between
a prebuilt multi-agent abstraction and a custom Workflow that runs each agent as a step. Reach for the
prebuilt agent/handoff pattern when the coordination is a standard hand-off (router → specialist →
back). Drop to a custom Workflow when the coordination itself is non-standard — conditional routing on
agent output, human gates between agents, partial-result joins, or per-agent retry policies. The
Workflow gives you the seams to insert that logic; a prebuilt orchestrator hides them. Start with the
prebuilt path and graduate to a Workflow only when you hit a coordination need it cannot express.

## The core model

The shift from a DAG to an event graph is the whole point: in a DAG you declare *edges* and the
engine walks them; in a Workflow you declare *what each step consumes and produces*, and the engine
derives the edges. That inversion is why control flow lives in ordinary Python/TS — an `if` that
returns a different event *is* a branch, an event that routes backward *is* a loop. There is no
separate flow-definition language to keep in sync with the code.

- **Events are the unit of flow.** Subclass the event base for type safety; each carries arbitrary
  (typically Pydantic) payload. The set of event types a step accepts/returns defines its place in
  the graph.
- **Entry and exit are special events.** A start event begins the run (its attributes come from the
  `.run(...)` kwargs); a stop event ends it and carries the result. Subclass them for typed I/O.
- **Steps consume one event, emit the next.** A step that returns event type `B` is automatically
  wired to whatever step accepts `B`. No manual edge declaration.
- **Validation is upfront.** The framework checks the event graph is connected (no orphan/unreachable
  steps, every emitted event has a consumer) *before* running — catching wiring bugs early.
- **`Context` is the shared store + run handle.** Use it for cross-step state, for emitting events to
  the stream, and for the human-in-the-loop pause/resume. State lives in a typed store on the Context
  rather than in step return values when many steps need it.

## Branching, looping, fan-out

- **Branch**: a step emits one of several event types; the matching downstream step runs. Conditional
  routing is just an `if` choosing which event to return.
- **Loop / retry**: a step emits an event that routes *back* to an earlier step (e.g. a validation
  step re-emits the work event on failure). Loops are cycles in the event graph, expressed naturally.
- **Fan-out (parallel)**: emit multiple events from one step to run branches concurrently; a
  **collector/join** step waits for the expected number of results before proceeding. The Context
  provides the primitive to gather N events of a type. Prefer this over manual `asyncio` plumbing.

Because steps fire when their input event arrives — not in source order — concurrency falls out for
free: two steps consuming the same event run in parallel, and independent branches advance
independently. Sequencing comes from the event dependencies, not from where a method sits in the
file. Cap any loop with a counter held in Context so a step that keeps re-emitting its own trigger
cannot run forever.

## Human-in-the-loop (HITL)

The workflow **pauses, surfaces a request to the outside world, and resumes** when a response comes
back — without losing in-flight state. Conceptually: a step emits an "input required" event; the
runtime surfaces it (via the stream); external code submits a "human response" event; a waiting step
resumes. Because Context holds the state, the pause can span an HTTP round-trip or longer. This is the
backbone of approvals and clarification loops. Exact event/method names drift — fetch live.

## Streaming

A run exposes a **stream of events** as it executes. Steps write progress events to the stream via the
Context; the caller iterates the handler's stream to drive a UI (token deltas, tool-call traces, step
boundaries). The final result still comes from awaiting the run handle. Use streaming whenever a user
is watching — do not block on the final stop event for long runs.

## Durability

Runs can be **checkpointed** so a crashed or restarted process resumes mid-flight rather than
restarting. In-memory state is the default; durable backends (e.g. SQLite for single-process, a
Postgres/DBOS-backed store for distributed production) persist handler state and enable automatic
recovery. Treat durability as a deployment-time decision: develop in-memory, add a persistence backend
when the workflow is long-running or must not lose progress. The *serving* layer that wires this up is
`llamaindex-deploy` territory — here, just design steps to be **idempotent and resumable** (no
side effects that break on replay).

## The serving model (brief — defer the how-to)

A finished Workflow is exposed over HTTP by a workflow **server** that turns each workflow into REST
endpoints supporting synchronous run and async (handler-id + poll) execution, plus event injection for
HITL and a debugging UI. Know that this path exists so you design workflows to be serializable and
resumable — but for the actual `llamactl` / server setup, hand off to **`llamaindex-deploy`**.

## Minimal examples (illustrative — fetch exact signatures live)

Python:

```python
# Standalone package surface (verify install + imports live — see "Fetching facts").
# A compatibility path under llama_index.core.workflow also exists.
from workflows import Workflow, step
from workflows.events import Event, StartEvent, StopEvent

class Retrieved(Event):
    text: str

class RAGFlow(Workflow):
    @step
    async def retrieve(self, ev: StartEvent) -> Retrieved:
        return Retrieved(text=do_retrieve(ev.query))

    @step
    async def synthesize(self, ev: Retrieved) -> StopEvent:
        return StopEvent(result=llm_answer(ev.text))

# result = await RAGFlow(timeout=60).run(query="...")
```

TypeScript (`@llamaindex/workflow`): define a workflow, register handlers keyed on event types, and
run it the same event-in/event-out way. Confirm package and API live — the TS surface evolves.

## Common pitfalls (see references for the full taxonomy)

- Using a Workflow for a linear single-shot query — over-engineering; use a query engine.
- Holding state in step return values when many steps need it — put it in the Context store instead.
- A step emits an event no step consumes (or vice versa) — caught by upfront validation; fix the
  event types.
- Forgetting the join/collector step on a fan-out — the workflow stalls waiting for results.
- Non-idempotent side effects that break on checkpoint replay.
- Blocking on the final result when a UI needs streamed progress.

## Fetching facts (do not hardcode)

Treat as **live lookups**, never memorized: package names/versions, the exact event class and
`Context`/step method names, the HITL event names, and streaming/checkpointer APIs — they drift across
releases. Fetch via:

- **Docs MCP** (wired, no auth): `mcp__plugin_llamacloud_docs__search_docs` for concepts,
  `mcp__plugin_llamacloud_docs__read_doc` for a known page, `mcp__plugin_llamacloud_docs__grep_docs`
  for an exact symbol.
- **Clean-markdown fallback:** append `index.md` to any `https://developers.llamaindex.ai/...` page
  URL and `WebFetch` it.

Anchor pages: `/python/llamaagents/workflows/` (model + examples: branching, looping, HITL,
streaming, checkpointing), `/python/llamaagents/workflows/deployment/` (serving model — defer the
how-to), and the TypeScript `@llamaindex/workflow` docs for the TS surface.

## Additional resources

- **`references/workflow-patterns.md`** — when-to-use-a-Workflow decision table, event/step design
  patterns, state-vs-return-value calls, HITL and streaming decision guidance, durability/idempotency
  rules, the build/test sequence, and the full pitfall taxonomy.
