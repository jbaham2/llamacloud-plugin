# Workflow Patterns — Durable Judgment

Design depth for LlamaIndex Workflows. SKILL.md has the model; this file has the decisions.
For exact signatures (event classes, `Context`/step methods, HITL event names, checkpointer API),
fetch live — never quote them from here.

## When to use a Workflow (full table)

| Situation | Use | Why |
|---|---|---|
| Single query → answer, no branching | Query engine / chain (`llamaindex-framework`) | Orchestration overhead with no benefit |
| Linear pipeline, fixed steps, no loops | Chain / `QueryPipeline` | A Workflow buys nothing over a straight pipe |
| Branch on intermediate result | **Workflow** | Conditional event routing is the native idiom |
| Loop / retry until valid | **Workflow** | Cycles in the event graph are natural |
| Fan-out parallel + join | **Workflow** | Built-in collect-N primitive beats manual asyncio |
| Pause for human, then resume | **Workflow + HITL** | Only Workflows pause/resume cleanly across a round-trip |
| Must survive crash/restart | **Workflow + durability** | Checkpointing resumes mid-flight |
| Stream progress to a UI | **Workflow streaming** | Emits events as it runs |
| One prebuilt agent call | The agent directly (`FunctionAgent` etc.) | Wrap in a Workflow only to orchestrate around/between agents |
| Multiple cooperating agents / router | **Workflow** | Orchestrate agents as steps |

Rule of thumb: if you can draw the logic as a straight line, you do not need a Workflow. The moment
the line forks, loops, joins, pauses, or must be observed/resumed — use one.

## Event & step design

- **Model the data, not the steps.** Name events for the *data they carry* (`Retrieved`, `Reranked`,
  `Validated`), not the step that made them. Steps are wired by event type, so good event types =
  clear graph.
- **One responsibility per step.** A step consumes one event and emits the next. Avoid steps that do
  retrieve+rerank+synthesize — split them so branches and retries can hook between.
- **Typed start/stop events** for typed inputs/outputs. The start event's fields come from `.run()`
  kwargs; the stop event carries the final result.
- **Let upfront validation work for you.** Every emitted event needs a consumer and every step needs a
  reachable input. If validation complains about orphan/unreachable steps, the event types are wrong —
  fix the types, don't suppress the check.
- **Set a timeout** on long runs so a stuck step fails loudly instead of hanging.

## State: Context store vs. step return values

- Pass data **via the event** when it flows to exactly the next step.
- Put data in the **Context store** when several non-adjacent steps need it, or when a loop must
  accumulate across iterations (counters, partial results, conversation memory).
- Do **not** smuggle large shared state through chains of event payloads — it couples unrelated steps.
- Context state is per-run; treat it as the run's working memory.

## Branching, looping, fan-out

- **Branch**: return one of several event types from a step (an `if`/`match` choosing the event). Each
  target step declares the event it accepts.
- **Loop**: emit an event that routes back to an earlier step. Guard with a counter in Context to cap
  iterations — uncapped loops are the #1 way to burn tokens/time.
- **Fan-out**: emit N events of the same type to run branches concurrently. A **collector step** uses
  the Context's gather-N primitive to wait for all results before emitting downstream. Forgetting the
  collector (or miscounting N) **stalls the workflow** — the run hangs waiting for results that the
  join never receives.

## Human-in-the-loop

- Shape: a step emits an *input-required* event; the runtime surfaces it (typically via the stream);
  external code submits a *human-response* event; a waiting step resumes with the answer.
- State survives the pause because it lives in Context — design the pause point so everything needed to
  resume is already in Context, not in local variables.
- HITL pauses can span an HTTP round-trip or hours. That makes **durability** relevant: if the process
  may restart during the wait, checkpoint so the pending run survives.
- Use it for approvals, clarifying questions, tool-use confirmation. Don't use it for things the LLM
  can decide — every pause is latency and a UX cost.

## Streaming

- Steps write progress events to the stream via Context; the caller iterates the run handle's stream.
- Stream when a human is watching (token deltas, tool traces, step boundaries) or when steps are slow.
- The streamed events are *intermediate*; the final result still comes from awaiting the run handle —
  don't conflate the two.
- Define explicit progress event types rather than streaming raw internals, so the UI contract is
  stable.

## Durability & idempotency

- **Default is in-memory.** Add a persistence backend only when the run is long-lived, spans a HITL
  pause across restarts, or must not lose progress.
- Backends scale with need: SQLite for single-process; a Postgres/DBOS-backed store for distributed,
  production, auto-recovering deployments.
- **Design steps to be idempotent and replay-safe.** A checkpoint may re-run a step on recovery — so a
  step that, e.g., charges a card or sends an email must guard against double execution (check-then-act,
  idempotency keys). Side effects that break on replay are the subtle durability bug.
- Keep state serializable (Pydantic-friendly) so it can be checkpointed.
- Wiring the persistence backend into a *served* workflow is `llamaindex-deploy`'s job; here, ensure the
  workflow is *capable* of being made durable (serializable state, idempotent steps).

## Build & test sequence

1. Sketch the **events** (the data that flows) and which step emits/consumes each.
2. Implement steps in-memory, single-process; run end-to-end on a happy path.
3. Let upfront validation confirm the graph is connected; fix event-type mismatches.
4. Add branches/loops; cap loops with a Context counter.
5. Add fan-out + a collector; verify the join count.
6. Add streaming if a UI consumes progress.
7. Add HITL if a human gates a step.
8. Add durability last, once the logic is stable — then audit every side-effecting step for replay
   safety.
9. Hand off serving to `llamaindex-deploy`.

## Pitfall taxonomy

| Pitfall | Symptom | Fix |
|---|---|---|
| Workflow for a linear single query | Boilerplate, no branching used | Use a query engine (`llamaindex-framework`) |
| Orphan/unreachable step | Validation error before run | Align emitted/consumed event types |
| Emitted event with no consumer | Validation error / dead branch | Add the consuming step or drop the event |
| Fan-out without collector / wrong N | Run hangs forever | Add a collector step; gather the exact count |
| Uncapped loop | Runaway tokens/time | Cap with a Context counter + exit condition |
| State in event payloads across many steps | Tight coupling, fragile graph | Move shared state to the Context store |
| Non-idempotent side effect | Double charge/email on checkpoint replay | Guard with idempotency keys / check-then-act |
| Blocking on final result for a live UI | UI shows nothing until done | Stream intermediate events |
| Local-variable state at a HITL pause | Lost state after resume/restart | Keep resume state in Context; checkpoint if restart-prone |
| Non-serializable Context state | Checkpointing fails | Use serializable (Pydantic) state |

## Boundaries

- Serving the workflow as an HTTP API (workflow server, `llamactl`, persistence wiring) →
  `llamaindex-deploy`.
- Plain RAG component wiring (loaders, indexes, retrievers, query engines, prebuilt agents) →
  `llamaindex-framework`.
- This skill: the event-driven orchestration model and the judgment around it.
