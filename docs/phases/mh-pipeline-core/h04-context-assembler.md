# H4 — Context assembler and prompt caching

**Milestone:** MH Pipeline core · **Status:** ⬜ Not started · **From harness:** F4, F20 · **Depends on:** [H2](h02-provider-adapters.md), [H3](h03-turn-store-tracer.md) · **Unblocks:** [H12](h12-error-recovery-reasoning.md), [H13](h13-compaction-tool-disclosure.md), [F12](../m02-local-knowledge/f12-quick-depth-e2e.md)

[← Back to plan](../PLAN.md)

## Goal

Build each model request from its parts — system prompt, tool schemas, history, retrieved passages and injected notes — within a token budget. Order those parts so that provider prompt caching works. Research runs make many calls with the same system prompt and tools, so a stable prefix saves real money and latency.

## Scope

**In scope**

- A `ContextAssembler` that turns a `TurnContext` into a `ChatRequest` (H2).
- Token estimation per model, using a tokenizer where one is available and a character heuristic otherwise.
- A budget derived from the model's context window minus a reserve for the reply.
- Trimming oldest-first, which never drops the system prompt, the current user message, or pinned items.
- Fixed prefix order:
  1. System prompt.
  2. Tool schemas.
  3. Stable project context.
  4. History.
  5. Volatile notes, last.
- Anthropic cache breakpoints. Cache read and write stats go in the trace.
- Delimiting untrusted content, such as fetched passages and notes, as data blocks.

**Out of scope**

- Summarizing dropped turns, and clearing stale tool results → [H13](h13-compaction-tool-disclosure.md).
- Deferred tool schemas and `tool_search` → [H13](h13-compaction-tool-disclosure.md).
- Retrieval of passages → [F9](../m02-local-knowledge/f09-passage-store.md).
- Thinking budgets and keeping thinking blocks → [H12](h12-error-recovery-reasoning.md).

## Design notes

- Context assembler: builds each model request from the system prompt, history, reminders and tool schemas. It budgets tokens, trims oldest-first, and then summarizes dropped turns. (Harness: Core services)
- The harness Done-when: a long history trims oldest first and never drops the system prompt. (Harness: Feature breakdown F4)
- Prompt caching: keep the system prompt and tool schemas as a fixed prefix, append volatile notes last, and mark cache breakpoints for Anthropic. (Harness: Production harness behaviors)
- The harness Done-when for caching: the second turn reports cache hits, and injected notes never break the prefix. (Harness: Feature breakdown F20)
- `Inject` notes from counselors and `Annotate` notes from input stages are appended to the next model request. (Harness: Stage 1; Stage 2)
- Prompt-injection guard on fetched text: fetched content is data, never instructions. (Design: Research pipeline › How the chat harness stages map)

```toml
[cache]
enabled = true          # provider-specific breakpoints
```

## Implementation sketch

Suggestions, not decisions:

1. Represent a request as ordered `Segment`s with a `stability` tag: `Fixed`, `Session` or `Volatile`. The assembler sorts by stability and places a breakpoint after the last `Fixed` segment and after the last `Session` segment.
2. Wrap untrusted text in tagged blocks such as `<source id="p:123" trust="untrusted">…</source>`, and tell the model in the system prompt how to treat them.
3. Measure the budget as the context window minus `max_tokens` minus a safety margin. Drop whole history turns, oldest first, until the request fits. Record each drop as a trace event.
4. Use OpenAI's automatic caching as is, and just keep the prefix stable. Local endpoints ignore cache hints.
5. Add a test: two successive calls with different volatile notes produce byte-identical prefixes.

## Acceptance criteria

- [ ] A long history trims oldest-first and never drops the system prompt or the current message. A test proves it.
- [ ] The second call of a research step reports cache hits against the Anthropic adapter, recorded in the trace. Use a mock or a real API.
- [ ] Adding or changing an injected note leaves the cached prefix byte-identical.
- [ ] Fetched or note content is always wrapped as an untrusted data block.
- [ ] The trace shows the token estimate, the budget, and what was trimmed.

## Open questions

- Should retrieved passages count as `Session` or `Volatile`? It depends on whether they stay stable across a sub-question's steps. Measure the cache hit rate once F12 runs.
- Tokenizers for local models differ. A heuristic plus a safety margin may be enough. Confirm against LM Studio.
