# H13 — Context compaction and progressive tool disclosure

**Milestone:** MH Pipeline core · **Status:** ⬜ Not started · **From harness:** F19, F24 · **Depends on:** [H4](h04-context-assembler.md), [H5](h05-tool-loop.md) · **Unblocks:** [F24](../m06-async-research/f24-planner-checkpoint.md), [F31](../m08-project-chat/f31-project-chat-loop.md)

[← Back to plan](../PLAN.md)

## Goal
Keep long agentic turns and long chat threads working without overflowing the context window. When a request nears its limit, compaction clears stale tool results first, then summarizes the oldest turns. Progressive disclosure keeps the first request small by sending only core tool schemas, and loads the rest on demand. Project chat threads and long research steps both need this. Compaction also maintains a running **thread brief** for each chat thread: a short summary of the thread's important points. Research planning reads the brief, so ongoing chat feeds into research (PLAN.md decision D11).

## Scope
**In scope**
- Compaction triggered at a fraction of the model's context window.
- Clearing tool results older than N turns, replaced by a short placeholder.
- Summarizing the oldest turns into one hidden message, using the summarizer role.
- Never dropping the system prompt, the pinned scope, or the current turn.
- Progressive disclosure: deferred tools listed by name only, plus a `tool_search` tool that loads their schemas.
- A trace of what was cleared, the summary text, and the token savings.
- A persisted **thread brief** per chat thread (D11). It holds the important points: decisions, constraints and preferences the user stated, open questions, and the claim and passage ids discussed. It is updated whenever the thread compacts, and also after every N turns. Other features read it through `thread_brief(thread_id)`.

**Out of scope**
- Basic trimming and token budgeting → [H4](h04-context-assembler.md).
- Prompt-cache breakpoints → [H4](h04-context-assembler.md). Compaction must respect them.
- Skill files. The harness loads them on demand, but Searchy has no skills concept. Domain profiles ([F41](../m10-profiles-media/f41-profile-system.md)) are configuration, not prompt files.
- Chat tools themselves → [F31](../m08-project-chat/f31-project-chat-loop.md) and [F33](../m08-project-chat/f33-scoped-chat.md).
- Injecting thread briefs into planner and research-agent context → [F24](../m06-async-research/f24-planner-checkpoint.md). This feature only produces and stores the brief.

## Design notes
**Context compaction** (Harness: Production harness behaviors): near a token threshold, clear stale tool results first, then summarize the oldest turns into one hidden message. It teaches what to keep versus drop.

**Progressive disclosure** (Harness: Production harness behaviors): send only core tool schemas. A `tool_search` tool loads deferred schemas. The trade is a lookup step for a smaller, cheaper context.

```toml
[tools]
deferred = ["query_dataset", "make_chart", "make_table"]   # loaded via tool_search

[context]
compact_at = 0.8                     # fraction of the context window
clear_tool_results_older_than = 6    # turns
summary_role = "summarizer"          # Searchy routes by role (F8)
```

**Chat feeds research** (PLAN.md D11): project chat is part of ongoing research. Chat threads are compacted, and their important points are kept as a running summary. That summary is injected into the planner's and research agents' context. Research steps otherwise start from a fresh context, so compaction inside research steps is a safety net, not the normal path.

**Context assembler basics** (Harness: Core services): oldest-first trimming, then summarization of dropped turns.

**Searchy chat tools** (Design: Project chat): `search_claims`, `get_passage`, `get_source`, `list_contradictions`, `list_gaps`, `get_cluster`, `query_dataset`, `make_chart`, `make_table`, `save_output` and `start_research`. That is a lot of schemas, which makes deferral worthwhile.

**Citations survive compaction** (Design: Research pipeline; Project chat): answers cite passage ids. A summary must keep the passage ids it refers to, or later turns can't cite them.

## Implementation sketch
Suggestions, not decisions.
- A `Compactor` called by the [H4](h04-context-assembler.md) assembler before each request when the estimated tokens exceed `compact_at`.
  - Pass 1 replaces old tool-result bodies with `[cleared: <tool> result, N tokens, ids: …]`.
  - Pass 2 summarizes the oldest turns, if still over budget.
- Store the summary as a hidden message linked to the message range it replaces. The full history stays in SQLite, so the thread view is unchanged.
- The summarizer prompt must preserve passage ids, claim ids and numbers verbatim. Test that ids survive.
- Compact only after the cache breakpoint's prefix, so caching from H4 still hits.
- Thread brief: a `thread_briefs (thread_id, project_id, brief_md, cited_ids_json, updated_at, through_message_id)` table. Update it with the summarizer role, giving it the previous brief plus the new turns, so the brief stays bounded (for example, ≤ 400 tokens). Its sections are Decisions, Constraints and preferences, Open questions, and Key claims (with ids). It must keep ids verbatim, like summaries do.
- `tool_search(query)` matches names and descriptions in the registry from [H5](h05-tool-loop.md). Matched schemas are added to the tool list for the rest of the turn.
- Steps:
  1. Token estimation hooks.
  2. Clearing tool results.
  3. Summarization.
  4. The `tool_search` tool and deferral.
  5. Trace events with token counts before and after.

## Acceptance criteria
- [ ] A chat thread pushed past `compact_at` keeps working. The trace shows what was cleared and the summary text.
- [ ] The system prompt, the scope and the current turn are never compacted.
- [ ] Passage ids cited in compacted turns still appear in the summary.
- [ ] A deferred tool is absent from the first request, found through `tool_search`, then called. The trace shows the token savings.
- [ ] Compaction doesn't change the cached prefix, so the next request still reports cache hits.
- [ ] After a thread compacts, its brief is updated and lists the user's stated constraints and the claim ids discussed. `thread_brief(thread_id)` returns it.
- [ ] The brief stays under its token cap across a 50-turn thread.

## Open questions
- Resolved (D11): research steps start from a fresh context, plus the injected thread briefs. Compaction mainly serves chat, and it produces the briefs research reads.
- Which threads' briefs are relevant to a given question is decided in [F24](../m06-async-research/f24-planner-checkpoint.md).
