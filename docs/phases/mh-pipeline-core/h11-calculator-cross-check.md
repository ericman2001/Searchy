# H11 — Calculator tool and computed-answer check

**Milestone:** MH Pipeline core · **Status:** ⬜ Not started · **From harness:** F17, F18 · **Depends on:** [H5](h05-tool-loop.md), [H7](h07-input-stages.md), [H8](h08-counselors.md), [H9](h09-turn-end-validation.md) · **Unblocks:** [F15](../m03-ledger-verification/f15-number-cross-check.md)

[← Back to plan](../PLAN.md)

## Goal
Build the harness's worked example end to end: a question with arithmetic is steered to a calculator, wrong tools are blocked, and the answer is checked by the harness itself. It proves that all three stages cooperate in one turn. It also builds the cross-check machinery that Searchy's number verification ([F15](../m03-ledger-verification/f15-number-cross-check.md)) reuses: the harness reads a number from its own source and compares it, instead of trusting the model.

## Scope
**In scope**
- A safe `calculator` tool: a pure expression evaluator, never code execution.
- A `math` rules classifier with expression extraction into `labels.math.expr`.
- The `tool_hint` input stage, with a hidden note and an optional `force_tool`.
- The `tool_router` counselor, which denies listed tools when a label matches.
- The `computed_answer` turn-end validator, with the three-way check.
- A generic "cross-check" interface that F15 implements for passages.
- An end-to-end test using a scripted fake model.

**Out of scope**
- Re-reading numbers from passages in reports → [F15](../m03-ledger-verification/f15-number-cross-check.md).
- Numbers in charts read by the harness → [F34](../m08-project-chat/f34-outputs.md).
- A `judge` classifier for word problems → possible with [H10](h10-judges.md), but not required here.

## Design notes
**The flow** (Harness: Worked example: calculator-guided arithmetic)
1. A classifier spots arithmetic.
2. An input stage steers the model to the calculator.
3. A counselor blocks wrong tools.
4. The turn-end check verifies the number independently.

The example is chosen to be easy to trace. The same profile pays off on prompts like `48213 × 9177`.

```toml
[[classifier]]
name = "math"
kind = "rules"
labels = ["arithmetic", "none"]
extract = "expr"                # pulls "2+2" into labels.math.expr

[[input]]
stage = "tool_hint"
needs = ["math"]
when = "math:arithmetic"
note = "Use the calculator tool for all arithmetic. Do not compute mentally."
force_tool = "calculator"       # optional: provider tool_choice

[[counselor]]
stage = "tool_router"
hooks = ["before_tool_call"]
needs = ["math"]
when = "math:arithmetic"
deny = ["query_dataset", "web_search"]
deny_reason = "Use the calculator tool for arithmetic."

[[turn_end]]
stage = "computed_answer"
needs = ["math"]
when = "math:arithmetic"
require_tool = "calculator"
cross_check = "expr"            # harness evaluates labels.math.expr itself
```

**Design points** (Harness: Worked example › Design points)
- Hint versus force: a hidden note nudges, and `force_tool` sets the provider's `tool_choice`. Hinting is the better test of whether counselors and validators catch mistakes.
- **Three numbers must agree:** the calculator was called, the reply's number matches the tool result, and both match the harness's own evaluation of the extracted expression. This catches a model that calls `calc(2+3)`.
- Safe calculator: a pure evaluator such as `evalexpr`, which is deterministic and needs no sandbox.
- Classifier fallback: regex handles "what is 2+2", while word problems need a judge classifier.
- Trace to look for: the `math` label, the injected note, any denied call with its reason, the calculator call, and the three-way comparison.

**Searchy link** (Design: Source quality › Verification): "Numbers get the calculator-example treatment from the chat harness: the harness re-reads the number from the passage itself and compares."

## Implementation sketch
Suggestions, not decisions.
- A `CrossCheck` trait: `fn expected(&self, ctx) -> Vec<ExpectedValue>` plus a shared `compare(reply_numbers, tool_numbers, expected) -> CrossCheckReport`. The calculator implements `expected` by evaluating `labels.math.expr`. F15 implements it by parsing numbers from the cited passage span.
- Put the number parsing (separators, signs, percentages, units, tolerance) in one module, so F15 reuses it rather than re-deriving it.
- Check that `evalexpr` is still maintained when the feature starts. Disable its variable assignment and functions beyond arithmetic.
- The end-to-end test uses a scripted fake `ChatProvider` from [H2](h02-provider-adapters.md):
  - Step 1 calls the denied tool.
  - Step 2 calls `calc("2+3")`.
  - Step 3, after the retry, calls `calc("2+2")` and answers 4.

## Acceptance criteria
- [ ] A scripted fake model that calls a denied tool is denied, a wrong calculation triggers one retry, and the final reply passes the three-way check.
- [ ] `calc("2+2")` returns 4, `calc("48213*9177")` is exact, and input that isn't arithmetic is rejected rather than evaluated.
- [ ] The `math` classifier labels and extracts `2+2` and ignores text that isn't math.
- [ ] One trace shows the label, the injected note, the denied call, the calculator call and the three-way comparison.
- [ ] The `CrossCheck` interface and number parser are public to the pipeline crate and documented for F15.

## Open questions
- Number tolerance and rounding (for example, "about 4.3 million" against 4,312,550). The calculator wants exact matches, but F15's passages often round. The shared comparator needs a tolerance setting.
