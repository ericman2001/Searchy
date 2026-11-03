# H12 — Error recovery and reasoning management

**Milestone:** MH Pipeline core · **Status:** ⬜ Not started · **From harness:** F23, F25 · **Depends on:** [H2](h02-provider-adapters.md), [H4](h04-context-assembler.md) · **Unblocks:** [F8](../m02-local-knowledge/f08-model-routing.md), [F24](../m06-async-research/f24-planner-checkpoint.md), [F25](../m06-async-research/f25-fan-out-budget.md)

[← Back to plan](../PLAN.md)

## Goal
Make model calls survive the real world, and make thinking budgets a setting rather than an accident. Research runs last minutes to hours, so rate limits, overloads and malformed tool calls are certain. This feature retries them with backoff, falls back to another model, and repairs or rejects bad tool calls. It also gives each call a reasoning budget, which is the main knob depth presets turn.

## Scope
**In scope**
- Retrying 429, overload and transient network errors with configurable backoff.
- A fallback mechanism that moves to the next model when retries are exhausted.
- Repairing or rejecting malformed tool calls (bad JSON arguments, unknown tool), with the reason returned to the model.
- Recording which model actually ran, and why, in the trace.
- Reasoning management: a thinking budget per call, keeping or stripping thinking blocks across tool calls, and display modes.

**Out of scope**
- Which models are in a role's fallback chain, and per-role cost → [F8](../m02-local-knowledge/f08-model-routing.md). F8 configures the chains; this feature runs them.
- Which reasoning budget each role gets at each depth → [F24](../m06-async-research/f24-planner-checkpoint.md) and [F25](../m06-async-research/f25-fan-out-budget.md).
- Budget caps on spending → [F25](../m06-async-research/f25-fan-out-budget.md).
- Retrying failed validations → [H9](h09-turn-end-validation.md). That is a different loop.

## Design notes
**Error recovery** (Harness: Production harness behaviors): retry rate-limit and overload errors with backoff, fall back to another model, and repair or reject malformed tool calls. This is "where much of a harness's reliability actually lives."

**Reasoning management** (Harness: Production harness behaviors): set thinking budgets, choose whether thinking blocks are kept across tool calls, and choose how they are shown.

```toml
[reasoning]
budget_tokens = 4000
keep_across_tool_calls = true
display = "collapsed"   # hidden | collapsed | full

[recovery]
max_retries = 3
backoff_ms = [500, 2000, 8000]
fallback_model = "small"   # Searchy: a model alias or the next entry in the role's chain
```

**Searchy rules** (Design: Model routing › Rules)
- Fallback chains: rate limits or outages move to the next model in a role's list, and the trace and report record which model actually ran.
- Local-only data stays local. A fallback must never move a local-only call to a cloud model. If no local fallback remains, skip the step and flag it.

**Reasoning by depth** (Design: Depth presets)
- Quick: low everywhere.
- Standard: medium for the planner and judge.
- Thorough: high for the planner, judge and reviewer.

Extraction and summarizing run on small fast models at every depth.

## Implementation sketch
Suggestions, not decisions.
- A `RecoveringProvider` wrapper around any `ChatProvider` from [H2](h02-provider-adapters.md). It takes an ordered list of providers and a `RecoveryPolicy`. It classifies errors as retryable, fallback-worthy or fatal, and honours `Retry-After`.
- Add `TraceEvent`s for each attempt: model, error class, wait time and the final model used.
- The local-only guard: each call carries a `DataLocality` flag from `TurnContext`, and the wrapper filters out cloud providers before trying them. F8 sets the flag; this feature enforces it during fallback.
- Malformed tool calls:
  - Try a strict JSON parse, then a light repair (trailing commas, unquoted keys).
  - If repair fails, return a tool result with the reason, so the model can try again.
  - Never pass guessed arguments to a tool.
- Reasoning: add a `ReasoningConfig` to the request. The [H4](h04-context-assembler.md) assembler keeps or strips prior thinking blocks. Providers without thinking support ignore the budget, and the trace notes that.
- The web display: chat renders thinking as hidden, a collapsed `<details>` element, or full text.
- Steps:
  1. Error classification and retry.
  2. Fallback.
  3. The local-only guard.
  4. Tool-call repair.
  5. The reasoning config in the assembler and providers.
  6. Display modes.

## Acceptance criteria
- [ ] Against a fault-injecting mock provider, retries succeed. Exhausted retries fall back to the next model. Bad JSON arguments are rejected with a reason the model sees.
- [ ] The trace records every attempt and names the model that finally answered.
- [ ] A local-only call never falls back to a cloud provider. It is skipped and flagged instead.
- [ ] Each `keep_across_tool_calls` setting produces the expected request body.
- [ ] The chat view shows thinking as hidden, collapsed or full according to the setting.

## Open questions
- Should a fallback to a weaker model be allowed silently in thorough runs, or should it ask, given that reports must be reproducible? At minimum it is recorded on the run.
