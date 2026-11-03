# H9 — Turn-end validation, retry loop and streaming modes

**Milestone:** MH Pipeline core · **Status:** ⬜ Not started · **From harness:** F13, F16 · **Depends on:** [H5](h05-tool-loop.md), [H6](h06-stage-runner-classifiers.md), [F7](../m01-deployable-shell/f07-progress-notifications.md) · **Unblocks:** [H10](h10-judges.md), [H11](h11-calculator-cross-check.md), [H14](h14-interruption-branching-replay.md), [F12](../m02-local-knowledge/f12-quick-depth-e2e.md), [F14](../m03-ledger-verification/f14-verification-stage.md)

[← Back to plan](../PLAN.md)

## Goal
Review every draft before anyone sees it, and send the model back with feedback when the draft fails. This is the machinery that Searchy's claim verification runs on. It also decides how text reaches the browser: held back until checked, or streamed live and corrected afterwards. This feature builds the retry loop, the deterministic starter validators and the streaming modes over SSE.

## Scope
**In scope**
- The turn-end runner. When the model stops calling tools, validators run in profile order on the draft.
- Outcomes: `Pass`, `Warn`, `Retry { feedback, spans }` and `Block`, plus the retry limit.
- Sentence-level retries (PLAN.md decision D10). The draft is split into numbered sentences. A `Retry` names the failing sentences, and only those are rewritten and re-checked.
- Starter validators: format, grounding (deterministic), promise check, and length/policy.
- Streaming modes: `buffered`, `stream_then_verify` and `off`, delivered through the SSE channel from [F7](../m01-deployable-shell/f07-progress-notifications.md).
- Storing retry feedback as hidden messages, and storing every draft, not just the last one.

**Out of scope**
- The completeness judge and other model-based validators → [H10](h10-judges.md).
- The computed-answer three-way check → [H11](h11-calculator-cross-check.md).
- Claim-to-passage verification and citation completeness → [F14](../m03-ledger-verification/f14-verification-stage.md), built as validators on this runner.
- Number re-reads from passages → [F15](../m03-ledger-verification/f15-number-cross-check.md).
- Replaying a turn under another profile → [H14](h14-interruption-branching-replay.md).

## Design notes
**Inputs each validator sees** (Harness: Stage 3): the original user message, the transformed message, the draft reply, and the turn's tool calls and results.

**Outcomes** (Harness: Stage 3 › Outcomes)
- `Pass`: deliver the reply.
- `Warn`: deliver with a visible flag, such as "unverified claim".
- `Retry { feedback }`: the feedback is added as a hidden message and the loop resumes. It counts against the retry limit.
- `Block`: replace the reply with a fallback message and the reason.
- Retry limit reached: deliver the last draft with a `Warn`. Never loop forever. The harness default is 2.

**Starter validators** (Harness: Stage 3 › Starter validators)
1. Format: if JSON, a table or code was asked for, check that it parses.
2. Grounding: numbers and file names in the reply should appear in tool results. Flag ones that don't.
3. Promise check: catch replies that say "I'll now do X" but end without doing it.
4. Length and policy: simple caps and banned-content rules.

**Streaming modes** (Harness: Stage 3 › Streaming modes)
- Buffered: show a "checking" indicator and reveal only validated text.
- Stream then verify: stream live, then validate. On failure, mark the reply superseded and stream the retry below it.
- Off: no turn-end stage, for comparison.

**Sentence-level retry** (PLAN.md D10): Searchy retries only the failing sentences, not the whole draft. The harness's `Retry { feedback }` gains a list of spans:

```rust
Retry { feedback: String, spans: Vec<SpanRef> }   // empty spans = whole draft
struct SpanRef { sentence_id: u32, reason: String }
```

Validators that judge the whole draft, such as format and promise check, return an empty `spans`. Per-sentence validators, such as grounding and F14's citation check, name each failing sentence.

**Searchy mapping** (Design: Research pipeline): step 7 sends failures back to Synthesize with reasons, up to the retry limit, then ships with visible flags. The retry limit comes from the depth preset: Quick 1, Standard 2, Thorough 3 (Design: Depth presets). Chat streams over SSE into an htmx-swapped message list (Design: Tech stack).

```toml
[[turn_end]]
stage = "format"

[[turn_end]]
stage = "promise_check"

[turn_end_policy]
max_retries = 2
streaming = "buffered"   # buffered | stream_then_verify | off
```

## Implementation sketch
Suggestions, not decisions.
- `TurnEndRunner::run(ctx) -> TurnEndOutcome`. It loops: tool loop ([H5](h05-tool-loop.md)), then validators, then either deliver or append feedback and loop again.
- Merge rule when several validators return a verdict: `Block` beats `Retry`, which beats `Warn`, which beats `Pass`. Feedback from every `Retry` is concatenated into one hidden message. Their spans are unioned, and any whole-draft `Retry` (empty spans) makes the whole retry a full redraft.
- Sentence segmentation: when the draft is produced, split it into sentences with stable ids. Citations such as `[p:123]` stay attached to their sentence. Store the segmented draft so the trace can show which sentences failed.
- Span retry:
  1. Send the model a hidden message listing only the failing sentences, each with its reason. The surrounding text is included as read-only context.
  2. Ask for replacements, keyed by `sentence_id`. The model may return an empty replacement to delete a sentence.
  3. Splice the replacements into the draft.
  4. Re-run per-sentence validators only on the replaced sentences, and whole-draft validators on the full draft.
  5. Each span retry counts once against the retry limit. When the limit is reached, the still-failing sentences are delivered with a per-sentence `Warn` flag, such as "unverified", instead of failing the whole reply.
- SSE events: `draft_started`, `token`, `checking`, `superseded`, `final`. Buffered mode sends only `checking` and `final`. Research runs use buffered mode, because reports are not streamed token by token.
- In stream-then-verify mode, a span retry swaps only the affected sentences. Each one gets a "revised" marker. A whole-draft retry keeps the old draft with a "replaced" label and streams the new one below it.
- Grounding validator: extract numbers with a regex and normalise thousands separators and units, then look them up in the tool-result text. F15 replaces this with passage-aware checks.
- Steps:
  1. Write the runner and the merge rule.
  2. Write the format, promise, and length validators.
  3. Write the grounding validator.
  4. Wire the SSE modes.
  5. Persist each draft and its verdicts.

## Acceptance criteria
- [ ] A draft with invalid JSON triggers exactly one retry. When the limit is reached, the last draft is delivered with a visible warning.
- [ ] Retry feedback is stored as a hidden message and appears in the trace, not in the user's view.
- [ ] Buffered mode sends no tokens to the browser until the validators pass.
- [ ] Stream-then-verify shows the first draft live, marks it superseded on failure, and streams the retry below it.
- [ ] The retry limit is configurable per profile, and the research run can override it per depth.
- [ ] A `Block` replaces the reply with the fallback message and its reason.
- [ ] A validator that fails sentence 3 of a 6-sentence draft causes only sentence 3 to be rewritten. Sentences 1–2 and 4–6 are byte-identical in the delivered reply, and the trace shows the span retry.
- [ ] When the retry limit is reached with one sentence still failing, the reply is delivered with that sentence alone flagged.

## Open questions
- Should retry feedback be visible to the user, or only in the trace? This is still open in the harness doc.
- Resolved (D10): retries target only the failing sentences, through `Retry { feedback, spans }`.
- Sentence splitting on Markdown (lists, tables, quotes) needs care. A table row or list item can count as one "sentence" unit.
