# H10 — Judge helper and judge classifiers

**Milestone:** MH Pipeline core · **Status:** ⬜ Not started · **From harness:** F15 (harness F14, script hooks, is cut by D12) · **Depends on:** [H2](h02-provider-adapters.md), [H8](h08-counselors.md), [H9](h09-turn-end-validation.md) · **Unblocks:** [F12](../m02-local-knowledge/f12-quick-depth-e2e.md), [F14](../m03-ledger-verification/f14-verification-stage.md), [F20](../m04-polite-web/f20-injection-counselor.md)

[← Back to plan](../PLAN.md)

## Goal
Make model-based checks a first-class, configurable part of the pipeline. A judge is a prompt, a model and an output format, so most of Searchy's checks can live in TOML: classifiers, the verifier, the drift counselor and the completeness validator. Searchy's verifier, question classifier and source-type classifier are all built on this.

## Scope
**In scope**
- A judge helper: one call to a small model with structured output, a hard timeout and parsing into a typed result.
- `judge` classifiers, alongside the `rules` classifiers from [H6](h06-stage-runner-classifiers.md).
- Prompt-defined judges as turn-end validators and as counselors.
- Two built-in judges: the completeness validator, and the drift counselor on `EveryNSteps`.
- Judge latency, tokens and the raw response in the trace.

**Out of scope**
- Choosing which model a judge uses → [F8](../m02-local-knowledge/f08-model-routing.md). Judges ask for a role (`classifier`, `verifier`), not a model id.
- Escalating to a larger model on uncertain verdicts → [F8](../m02-local-knowledge/f08-model-routing.md) and [F14](../m03-ledger-verification/f14-verification-stage.md).
- The citation verifier itself → [F14](../m03-ledger-verification/f14-verification-stage.md).
- The injection classifier → [F20](../m04-polite-web/f20-injection-counselor.md).
- Script hooks → cut (PLAN.md decision D12). Checks that need real code are built-in Rust stages, registered by name ([H6](h06-stage-runner-classifiers.md)).

## Design notes
**Two levels of configurability** (Harness: Configuration, minus script hooks per D12)
1. Built-in, parameterized: a Rust type registered by name. TOML picks it and sets its knobs.
2. Prompt-defined judges: a model-based check is a prompt, a model and an output format, so it lives entirely in TOML.

**Classifiers** (Harness: Configuration › Classifiers are a separate layer): each has a name, a kind (`rules` or `judge`; the harness's `script` kind is cut) and a label set. They run lazily, once per turn, and are cached on `TurnContext.labels`. Counselors can re-run a classifier on tool results.

```toml
[[classifier]]
name = "question_type"
kind = "judge"
role = "classifier"                 # Searchy routes by role, not model id
labels = ["factual", "comparative", "literature_review", "data_analysis"]
prompt = "Classify the research question into exactly one label."

[[turn_end]]
stage = "judge"
role = "verifier"
prompt = "Did the reply answer every part of the request? Return pass or fail and a reason."
timeout_ms = 3000
```

**Rules for model-based stages** (Harness: Stage 1 › Rules; Stage 2): use a small, fast model with a hard timeout. On timeout, `Pass` and log, rather than stall. Judges are opt-in per profile.

**Searchy uses for classifiers** (Design: Research pipeline): question type, source type, language, paywall detection and claim type. Rules first where possible (Design: Model routing › Roles).

**Why no script hooks** (D12; Design: Deployment › Isolation): Searchy ships as one binary, so hooks would just be built-in stages with extra plumbing. And a hub that runs arbitrary commands from a TOML file would undo the isolation design.

## Implementation sketch
Suggestions, not decisions.
- `Judge { prompt, role, output: OutputSchema, timeout }` with `async fn ask<T: DeserializeOwned>(&self, input) -> Result<JudgeResult<T>>`. Use the provider's JSON or tool-call mode from [H2](h02-provider-adapters.md) to get structured output. On a parse failure, re-ask once, then treat it as a timeout.
- A `JudgeResult` that carries a confidence or "unsure" flag. F8's verifier escalation needs that signal.
- Steps:
  1. Write the judge helper.
  2. Write judge classifiers.
  3. Write the judge validator and the drift counselor.

## Acceptance criteria
- [ ] Judges are opt-in per pipeline profile. Each judge call shows its latency, tokens and parsed result in the trace.
- [ ] A judge that times out returns `Pass`, logs the timeout, and doesn't stall the run.
- [ ] A `judge` classifier and a `rules` classifier can feed the same stage. A stage that `needs` both runs each once.
- [ ] Profile validation rejects a classifier with `kind = "script"` or any other unknown kind, with a message pointing to built-in stages.
- [ ] The completeness judge triggers a retry through [H9](h09-turn-end-validation.md) when the draft skips part of the request.

## Open questions
- How confidence or "unsure" is defined for judge output. F8's verifier escalation and PLAN's undefined-values list both depend on it.
