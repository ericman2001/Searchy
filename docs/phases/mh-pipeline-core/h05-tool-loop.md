# H5 — Tool registry, tool loop and parallel calls

**Milestone:** MH Pipeline core · **Status:** ⬜ Not started · **From harness:** F7, F22 · **Depends on:** [H2](h02-provider-adapters.md), [H3](h03-turn-store-tracer.md) · **Unblocks:** [H8](h08-counselors.md), [H9](h09-turn-end-validation.md), [H11](h11-calculator-cross-check.md), [H13](h13-compaction-tool-disclosure.md)

[← Back to plan](../PLAN.md)

## Goal

Give the pipeline its model/tool loop. The model asks for tools, the harness runs them, and the results go back until the model gives a final reply. Every research step and every chat turn runs on this loop, so tool definitions, argument validation and concurrent execution live here once.

## Scope

**In scope**

- A `Tool` trait and a `ToolRegistry`, with a name, a description, a JSON-schema for arguments and an async `execute`.
- Argument validation against the schema before a tool runs.
- The loop: call the model, run the requested tools, append the results, repeat until a final reply or a stop condition.
- Parallel tool calls: run the calls from one model step concurrently, up to `max_concurrent`, and return the results in request order.
- Per-call error handling. A failing tool returns an error result to the model and doesn't crash the turn.
- Hook points left for counselors: `BeforeModelCall`, `BeforeToolCall`, `AfterToolResult` and `AfterModelResponse`. They are no-ops until H8.
- A step limit and a hard stop, so the loop always ends.
- Starter test tools: `echo` and `fail`.

**Out of scope**

- Counselor logic → [H8](h08-counselors.md).
- Turn-end validation and retries → [H9](h09-turn-end-validation.md).
- Real research tools:
  - Retrieval → [F9](../m02-local-knowledge/f09-passage-store.md).
  - Fetching → [F17](../m04-polite-web/f17-fetch-proxy.md).
  - Datasets → [F26](../m06-async-research/f26-datasets-analyst.md).
  - Chat project tools → [F31](../m08-project-chat/f31-project-chat-loop.md).
- The calculator → [H11](h11-calculator-cross-check.md).
- Deferred schemas and `tool_search` → [H13](h13-compaction-tool-disclosure.md).
- Cancelling a running tool → [H14](h14-interruption-branching-replay.md).

## Design notes

- Tool registry: tool definitions, JSON-schema arguments and execution. (Harness: Core services)
- The harness Done-when: the model calls a tool, the result returns, and the loop ends on the final reply. (Harness: Feature breakdown F7)
- Parallel tool calls: run several calls from one model step concurrently, and return the results in request order. Each call gets its own counselor verdict, and partial failure is fine: one denied, two succeed. (Harness: Production harness behaviors)

```toml
[tools]
parallel = true
max_concurrent = 4
```

- Searchy tools by area. (Design: Project chat › How it works)
  - Retrieval: `search_claims`, `get_passage`, `get_source`.
  - Idea map: `list_contradictions`, `list_gaps`, `get_cluster`.
  - Data: `query_dataset`.
  - Outputs: `make_chart`, `make_table`, `save_output`.
  - Research: `start_research`.
- `start_research` proposes a run. It never starts one without the user's approval. (Design: Project chat)
- No MCP support; tools cover it. (Design: Goals and non-goals)
- Only the Fetcher touches the network. Tools that fetch must go through it, never through a direct HTTP client. (Design: Architecture)

## Implementation sketch

Suggestions, not decisions:

1. Define `trait Tool { fn spec(&self) -> ToolSpec; async fn execute(&self, args: Value, ctx: &ToolCtx) -> ToolResult; }`. `ToolCtx` carries the project scope, so a project-scoped tool can't read other projects.
2. Use the `jsonschema` crate for validation, and generate schemas with `schemars` from typed argument structs.
3. Write the loop as `async fn run_loop(ctx, provider, registry, hooks) -> LoopOutcome`. Use `futures::stream::iter(...).buffered(max_concurrent)` for ordered concurrent execution.
4. Store the tool calls and results on `TurnContext.tool_calls`, and trace each one with its latency.
5. Test it with H2's mock provider: a scripted model asks for three tools in one step, then gives a final reply.

## Acceptance criteria

- [ ] A scripted model calls a tool, the result returns, and the loop ends on the final reply.
- [ ] Three calls in one step run concurrently, and the model receives all three results in request order.
- [ ] Invalid arguments are rejected before the tool runs, and the model receives the validation error as the tool result.
- [ ] A tool that panics or errors produces an error result, and the turn continues.
- [ ] The loop stops at the step limit and records why in the trace.

## Open questions

- Tools that need a worker, such as `query_dataset` in the Analyst, are long-running jobs. The open question is whether the tool loop waits on a job (F6) or the tool proxies to a running worker. This ties to PLAN.md open questions 4 and 5.
