# H8 — Counselor hooks and starter counselors

**Milestone:** MH Pipeline core · **Status:** ⬜ Not started · **From harness:** F12 · **Depends on:** [H5](h05-tool-loop.md), [H6](h06-stage-runner-classifiers.md) · **Unblocks:** [F12](../m02-local-knowledge/f12-quick-depth-e2e.md), [F17](../m04-polite-web/f17-fetch-proxy.md), [F20](../m04-polite-web/f20-injection-counselor.md), [F21](../m05-source-quality/f21-source-classifier-trust.md), [F25](../m06-async-research/f25-fan-out-budget.md), [H10](h10-judges.md), [H11](h11-calculator-cross-check.md)

[← Back to plan](../PLAN.md)

## Goal
Let counselors watch the model/tool loop and steer it while it runs. A counselor can allow, deny, inject a hidden note, ask the user, or abort. This is the mid-turn layer that research runs depend on: the source-quality gate, the injection guard, the budget, the loop guard and domain politeness all ride on it. This feature builds the hook points and four starter counselors. Later features add the research-specific ones.

## Scope
**In scope**
- The five hook points inside the [H5](h05-tool-loop.md) tool loop.
- The `Counselor` interface and the counselor action enum.
- Binding counselors to hooks from the pipeline profile, with `needs` and `when` gating through [H6](h06-stage-runner-classifiers.md).
- Four starter counselors: permission, loop guard, budget and untrusted-content.
- Per-call verdicts when one model step makes several tool calls.
- `AskUser` becomes an Inbox item (PLAN.md decision D9). The step that asked pauses until someone answers, and other branches of the run keep going.
- A trace entry for every counselor action, including hidden notes.

**Out of scope**
- The model-based drift judge → [H10](h10-judges.md), which supplies the judge helper.
- Injection detection on fetched pages with a classifier → [F20](../m04-polite-web/f20-injection-counselor.md). It extends the untrusted-content counselor.
- Per-run money budgets, depth caps and the instance, user and project caps → [F25](../m06-async-research/f25-fan-out-budget.md). It extends the budget counselor.
- Source-quality gate and domain politeness → [F21](../m05-source-quality/f21-source-classifier-trust.md) and [F17](../m04-polite-web/f17-fetch-proxy.md).
- Citation chasing → [F36](../m09-review/f36-provenance-tiers.md).

## Design notes
**Hook points** (Harness: Stage 2 › Hook points)

| Event | Fires when | Typical uses |
| --- | --- | --- |
| `BeforeModelCall` | Each iteration, just before the request is sent | Inject reminders, trim context, enforce budget |
| `BeforeToolCall` | The model has requested a tool | Permission checks, argument validation, ask the user |
| `AfterToolResult` | A tool returned | Scan for injected instructions, truncate huge output |
| `AfterModelResponse` | A model step completed | Loop detection, drift check |
| `EveryNSteps` | Every N loop iterations | Model-judged progress check |

**Actions** (Harness: Stage 2 › Counselor actions)
- `Allow`: no change.
- `Deny { reason }`: the tool call doesn't run. The reason goes back to the model as the tool result, so it can adapt.
- `Inject { text }`: a hidden note is appended to the next model request (the "system reminder" pattern).
- `AskUser { question }`: pause for a yes/no or choice. In Searchy this always becomes an Inbox item, in chat turns and research runs alike. Only the step that asked pauses. (PLAN.md D9)
- `Abort { reason }`: end the turn and show the reason.

Counselors never edit model output directly. They influence it only through what goes into the next call.

**Starter counselors** (Harness: Stage 2 › Starter counselors)
1. **Permission:** per-tool allow, ask or deny rules from config.
2. **Loop guard:** the same tool with the same arguments 3 times in a row triggers an `Inject`, then an `Abort`.
3. **Budget:** caps tokens, tool calls and wall-clock time per turn.
4. **Untrusted-content:** marks tool results as data and warns the model if they contain instructions.

**Profile shape** (Harness: Configuration › Example profile)

```toml
[[counselor]]
stage = "permission"
hooks = ["before_tool_call"]
rules = { search_claims = "allow", start_research = "ask", fetch_url = "deny" }

[[counselor]]
stage = "untrusted_content"
hooks = ["after_tool_result"]
needs = ["injection"]

[[counselor]]
stage = "loop_guard"
hooks = ["after_model_response"]
repeat_limit = 3
```

**How Searchy uses counselors** (Design: Research pipeline › How the chat harness stages map): source-quality gate, prompt-injection guard on fetched text, budget, loop guard and domain politeness.

## Implementation sketch
Suggestions, not decisions.
- A `Counselor` trait alongside `Stage` in the pipeline crate from [H1](h01-pipeline-core-types.md). It has `hooks() -> &[HookPoint]` and `async fn on(&self, ev: &HookEvent, ctx: &mut TurnContext) -> CounselorAction`.
- Counselors for one hook run in profile order. The first `Deny` or `Abort` wins. Every `Inject` is collected into `ctx.pending_notes`.
- `AskUser` (D9):
  - It sets only the asking step, or the asking chat turn, to `awaiting_user`. Sibling sub-question steps in the same run keep running. The run itself stays `running` until every step is waiting or done.
  - It creates an Inbox item through [F7](../m01-deployable-shell/f07-progress-notifications.md), holding the question and its choices, and sends the usual push/ntfy notification.
  - Answering it re-queues the paused step. The answer is fed to the model as the counselor's result.
  - The paused loop state lives in SQLite, so a pending question survives a hub restart.
  - Suggested table: `pending_questions (id, project_id, run_id, step_id, thread_id, question, choices_json, answer, asked_at, answered_at, answered_by)`.
- Parallel calls from [H5](h05-tool-loop.md): call `BeforeToolCall` once per call, so one call can be denied while the others run.
- The budget counselor reads token counts from provider usage in [H2](h02-provider-adapters.md). It also exposes a `Budget` struct on `TurnContext` that F25 can extend with cost.
- Steps:
  1. Write the hook dispatcher.
  2. Write the permission and loop guard counselors.
  3. Write the budget counselor.
  4. Write the untrusted-content counselor, which wraps tool results in data delimiters.
  5. Wire `AskUser` to the UI.

## Acceptance criteria
- [ ] A denied tool call returns its reason to the model as the tool result, and a repeated identical call ends the step with an abort. Both are visible in the trace viewer.
- [ ] Profile validation rejects an unknown hook name or a counselor bound to a hook it doesn't support.
- [ ] When the model makes three calls in one step and one is denied, the other two still run.
- [ ] The budget counselor aborts a step that exceeds its tool-call cap and records which cap was hit.
- [ ] An `AskUser` pause shows in the Inbox, survives a hub restart, and resumes the loop when answered.
- [ ] In a run with two sub-question steps, an `AskUser` from one step pauses only that step. The other step runs to completion while the question is unanswered.
- [ ] `Inject` notes are hidden from the user but appear in the next request in the trace.

## Open questions
- Resolved (D9): `AskUser` becomes an Inbox item. The asking step pauses, and other branches continue.
- Should an unanswered question ever expire? The suggested default is no expiry. The run shows as partially waiting on the Run page.
- Whether the counselor trait and the stage trait should be one trait with a hook list, or two. The harness doc says stages share one interface, but counselors return a different action set.
