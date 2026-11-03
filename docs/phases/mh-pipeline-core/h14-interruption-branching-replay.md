# H14 — Interruption, branching and profile replay

**Milestone:** MH Pipeline core · **Status:** ⬜ Not started · **From harness:** F16, F21, F26 · **Depends on:** [H9](h09-turn-end-validation.md), [F7](../m01-deployable-shell/f07-progress-notifications.md) · **Unblocks:** [F31](../m08-project-chat/f31-project-chat-loop.md)

[← Back to plan](../PLAN.md)

## Goal
Give people control over turns that are already running, and over history. You can stop a streaming answer, cancel a running tool or research step, or queue a message while a turn is busy. Editing or regenerating a chat message forks the thread into a branch instead of overwriting it. Replay re-runs a recorded turn under a different pipeline profile, so you can compare traces side by side when tuning.

## Scope
**In scope**
- Stopping a streaming chat reply from the browser.
- Cancelling a running tool call or research step, leaving a consistent transcript.
- Queuing a message while a turn is in progress. It is answered next.
- Conversation branching for project-chat threads: a message tree, edit and regenerate, and a branch switcher.
- Replaying a stored turn under a different pipeline profile, with a side-by-side trace comparison page.

**Out of scope**
- The project chat itself → [F31](../m08-project-chat/f31-project-chat-loop.md).
- Cancelling or deleting whole research runs from the Run page → [F24](../m06-async-research/f24-planner-checkpoint.md)/[F25](../m06-async-research/f25-fan-out-budget.md). This feature provides the cancel mechanism they call.
- Branching research runs. Runs are not chat threads. A rerun is a new run ([F27](../m06-async-research/f27-watches.md)).

## Design notes
**Interruption and steering** (Harness: Production harness behaviors): the user can stop a stream, cancel a running tool, or queue a message mid-turn. It teaches what a turn is when it can be cut in half, and which state survives a cancel.

**Conversation branching** (Harness: Production harness behaviors): editing a past message or regenerating a reply forks history into a tree. Messages get a parent. Sessions get an active branch.

**Replay** (Harness: Feature breakdown › F16): the same input is replayed under two profiles, and the traces are compared side by side.

**Searchy constraints**
- All state lives in SQLite, so closing the tab or losing signal changes nothing (Design: Web interface › Async behavior). A stop must be an explicit action, not a dropped SSE connection.
- Chat threads are kept per project and are searchable later (Design: Project chat).
- Every action records who did it (Design: Teams and permissions › Attribution). Edits, regenerations and cancels are attributed.
- Viewers can read chat threads but not change them. Editors can chat (Design: Teams and permissions › Roles).

## Implementation sketch
Suggestions, not decisions.
- **Cancellation**
  - Each turn or step gets a `CancellationToken` (tokio-util), stored in a hub registry keyed by turn id.
  - A stop button posts `/turns/{id}/stop` through htmx. The loop checks the token between stream chunks and around tool calls.
  - Tools running in workers get a stop request through [F6](../m01-deployable-shell/f06-job-queue-launcher.md).
- **Consistent transcript**
  - A cancelled turn stores the partial draft marked `interrupted`.
  - Any tool call without a result gets a synthetic "cancelled by user" result, so the next request is still well-formed for the provider.
- **Queued messages**
  - A per-thread queue in SQLite.
  - When the current turn ends, the next queued message starts a new turn.
  - Optionally, a queued message is injected as a note at the next `BeforeModelCall` (steering), configurable.
- **Branching**
  - `messages.parent_id` and `threads.active_leaf_id`.
  - Edit creates a sibling user message. Regenerate creates a sibling assistant message.
  - The thread page shows "< 2/3 >" switchers rendered server-side.
- **Replay**
  - `replay(turn_id, profile)` re-runs with the stored transformed input and a new profile. It never re-fetches from the network; it uses archived tool results where possible.
  - The trace page from [H3](h03-turn-store-tracer.md) gets a two-column diff view.
- Steps:
  1. Cancel tokens and stop.
  2. Transcript repair.
  3. The queue.
  4. Branch schema and UI.
  5. Replay and diff page.

## Acceptance criteria
- [ ] A turn cancelled mid-stream or mid-tool leaves a consistent transcript, and a message queued during the turn is answered next.
- [ ] Closing the browser tab does not cancel a turn. Only the stop action does.
- [ ] Editing message 2 of a 5-message thread creates a branch. Both branches reload correctly after a hub restart.
- [ ] Regenerate creates a sibling reply, and the switcher moves between them.
- [ ] One input replayed under two pipeline profiles shows both traces side by side, with no new network fetches.
- [ ] Edits, regenerations and stops are attributed to the user who made them.

## Open questions
- Which branch does search over chat threads index: all branches, or only the active one?
- Should replay be admin-only? It can spend model budget without a plan checkpoint.
