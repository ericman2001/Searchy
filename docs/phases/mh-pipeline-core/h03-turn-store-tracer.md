# H3 — Turn store, tracer and web trace viewer

**Milestone:** MH Pipeline core · **Status:** ⬜ Not started · **From harness:** F3, F6 · **Depends on:** [H1](h01-pipeline-core-types.md), [F5](../m01-deployable-shell/f05-projects-data-model.md) · **Unblocks:** [H4](h04-context-assembler.md), [H5](h05-tool-loop.md)

[← Back to plan](../PLAN.md)

## Goal

Persist every message and every pipeline event, and show them in the browser. This makes the pipeline observable: you can open any run step or chat turn and see the exact request sent to the model, each stage's verdict, and the notes the user never saw. The harness's TUI trace pane becomes a hub page.

## Scope

**In scope**

- SQLite tables, added through F5's migrations:
  - Pipeline messages, with original and sent text and a hidden flag.
  - Turns, linked to a run step or a chat thread.
  - Trace events.
- A `Tracer` service that stages, counselors, providers and tools write to through `TurnContext`.
- A trace page in the hub, server-rendered with askama and htmx. It lists the turns for a run or thread. Each turn expands to its events, and each model call shows its raw request and response JSON.
- Live updates on the trace page over F7's SSE when that is available. Before then, a plain reload.
- Permissions: only project members can view a trace (F5 roles).

**Out of scope**

- Chat thread UI → [F31](../m08-project-chat/f31-project-chat-loop.md).
- Run progress page and Inbox → [F7](../m01-deployable-shell/f07-progress-notifications.md).
- Branching and the message tree → [H14](h14-interruption-branching-replay.md).
- Side-by-side replay comparison → [H14](h14-interruption-branching-replay.md).

## Design notes

- Conversation store: messages, tool calls and stage verdicts. SQLite, with an append-only message table that stores both original and sent text. (Harness: Core services) The harness's sessions table is dropped (PLAN.md D8). Each turn belongs to a run step (F5) or a chat thread (F31).
- Tracer: a structured event log for every stage, model call and tool call. A trace viewer shows the exact request sent at each step. (Harness: Core services)
- The transcript stores both the original message and what was actually sent, so every transformation is visible. (Harness: Stage 1 › Rules)
- All counselor activity, including notes the user never sees, appears in the trace view. Every label, its score and its latency appear too. (Harness: Stage 2; Configuration)
- `Message { id, turn_id, role, content_original, content_sent, hidden }`. `hidden` marks injected notes and validator feedback. The harness's `session_id` becomes the turn's owner, a run step or a chat thread (D8). (Harness: Core types)
- `TraceEvent { turn_id, stage, event, verdict, latency_ms, tokens }`. (Harness: Core types)
- Fallback chains must record which model actually ran, in both the trace and the report. (Design: Model routing › Rules)
- All state lives in SQLite on the host, so closing the tab changes nothing. (Design: Web interface › Async behavior)
- Every run, review and ledger edit records who did it. (Design: Teams and permissions › Attribution)

## Implementation sketch

Suggestions, not decisions:

1. Suggested tables:
   - `pipeline_turns (id, project_id, owner_kind, run_id, step_id, thread_id, profile_id, started_at, ended_at, outcome)`. `owner_kind` is `run_step` or `chat_thread`. A CHECK constraint requires exactly one of `(run_id, step_id)` and `thread_id`. There is no sessions table (D8).
   - `pipeline_messages (id, turn_id, role, content_original, content_sent, hidden, created_at)`
   - `trace_events (id, turn_id, seq, stage, event, verdict_json, payload_json, latency_ms, tokens_in, tokens_out, model, created_at)`
2. Store large raw request and response bodies in `payload_json`, compressed with zstd if size becomes a problem. Prune these after N days by config.
3. Make `Tracer` a trait with a SQLite implementation and an in-memory implementation for tests. Writes go through a bounded channel, so a slow disk never blocks a model call.
4. Page routes:
   - `/projects/:id/runs/:run/trace`
   - `/projects/:id/threads/:thread/trace`
   - `/trace/turn/:turn`, the detail view.
5. On a phone, the page shows a collapsed event list, with raw JSON behind a tap.

## Acceptance criteria

- [ ] Every model call in a test turn is visible as raw request JSON on the trace page.
- [ ] A transformed message shows both its original and its sent text.
- [ ] Hidden notes and validator feedback appear in the trace, marked as hidden.
- [ ] A user who isn't a member of the project gets a 404 on that project's trace URLs.
- [ ] Trace writes from a 100-event turn don't block the turn. Measure with the in-memory provider and a timing test.
- [ ] The trace page is readable at 380 px width.
- [ ] There is no sessions table. Inserting a turn with both a run step and a thread, or with neither, fails the CHECK constraint.

## Open questions

- Retention: raw payloads can grow fast on thorough runs. Pick a default retention, and decide whether a backup (F30) includes traces.
- Should viewers (F5's read-only project role) see hidden notes in traces? Proposed: editors and owners only.
