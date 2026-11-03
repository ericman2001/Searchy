# H6 — Stage runner, profile validation and rules classifiers

**Milestone:** MH Pipeline core · **Status:** ⬜ Not started · **From harness:** F9, F10 · **Depends on:** [H1](h01-pipeline-core-types.md) · **Unblocks:** [H7](h07-input-stages.md), [H8](h08-counselors.md), [H9](h09-turn-end-validation.md)

[← Back to plan](../PLAN.md)

## Goal

Run the stages a pipeline profile lists, in order, and handle their verdicts. Validate the profile at startup so mistakes fail early. Add the classifier layer: classifiers label the input once, and the stages read those labels. This is the engine that makes Searchy's research steps configurable in TOML rather than hard-coded.

## Scope

**In scope**

- A stage registry: built-in stages registered by name, so a profile can pick them and set their knobs.
- A stage runner that executes an ordered list of stages, applies the verdicts, enforces timeouts, and traces everything.
- Verdict handling:
  - The first `Block` stops.
  - `Transform` keeps the edits.
  - `Annotate` queues a hidden note.
  - `Warn` is recorded for display.
- `needs` and `when` gating.
- A classifier registry, with lazy evaluation of only the classifiers that enabled stages need. Results are cached on `TurnContext.labels` for the turn.
- The `rules` classifier kind: regex and keyword patterns, with optional field extraction.
- Startup validation of all pipeline profiles in a profiles folder.

**Out of scope**

- The `judge` classifier kind → [H10](h10-judges.md). The harness's `script` kind is cut (D12).
- Concrete input stages → [H7](h07-input-stages.md).
- Counselor event binding → [H8](h08-counselors.md).
- The turn-end retry loop → [H9](h09-turn-end-validation.md).
- Domain profiles and their `extends` overlays → [F41](../m10-profiles-media/f41-profile-system.md).

## Design notes

- Three levels of configurability. (Harness: Configuration)
  1. Built-in and parameterized: TOML picks a type registered by name and sets its knobs.
  2. Prompt-defined judges.

  The harness's third level, script hooks, is cut (D12).

  Real branching logic stays in code, never in config.
- Each classifier is declared once, with a name, a kind (`rules` or `judge`) and its label set. Stages list the classifiers they `need`. A stage can be gated with `when`, which is a simple label match like `"intent:sql"`, not an expression language. (Harness: Configuration › Classifiers)
- The runner computes only the classifiers that enabled stages need, lazily, and caches the results. Every label, its score and its latency are traced. (Harness: Configuration › Classifiers)
- The loader rejects these at startup: unknown stage names, a `needs` that names no classifier, and a `when` label outside that classifier's label set. (Harness: Configuration)
- Model-based stages have a hard timeout. On timeout they `Pass` and log, rather than stall. (Harness: Stage 1 › Rules)
- Searchy's classifiers cover question type, source type, language, paywall detection and claim type. (Design: Research pipeline › How the chat harness stages map)
- Custom profiles are validated at startup, "like the chat harness's stage profiles." (Design: Domain profiles › Auto-selection)

```toml
[[classifier]]
name = "secrets"
kind = "rules"
patterns = { api_key = "sk-[A-Za-z0-9]{20,}", conn_string = "Server=.*;Password=" }

[[input]]
stage = "sql_context"
needs = ["intent"]
when = "intent:sql"
```

## Implementation sketch

Suggestions, not decisions:

- H6 no longer waits on [H3](h03-turn-store-tracer.md). Trace through a small `TraceSink` trait with an in-memory implementation for tests, so H3's SQLite tracer can plug in later without changing the runner.
1. `StageRegistry: HashMap<&'static str, fn(&toml::Value) -> Result<Box<dyn Stage>>>`. Each stage parses its own knobs.
2. `ClassifierRegistry` builds classifiers from `[[classifier]]` blocks. `Labels::get_or_compute(name)` runs a classifier at most once per turn.
3. Wrap every `run` in `tokio::time::timeout`. On elapse, record `Pass` with a `timed_out` flag.
4. `ProfileLoader::load_dir("profiles/pipeline")` returns all errors at once, with the file and line where possible. The hub refuses to start if any profile is invalid.
5. Test with fake stages that return each verdict.

## Acceptance criteria

- [ ] Fake stages prove that `Block` stops the chain, `Transform` edits persist, and a timeout yields `Pass`. A bad profile fails at hub startup with a clear error.
- [ ] Two stages that need one classifier cause it to run exactly once. The trace shows the label, the score and the latency.
- [ ] A stage whose `when` doesn't match is skipped, and the trace records it as skipped.
- [ ] Each validation error has a test: an unknown stage, a missing classifier, and a `when` label outside the label set.
- [ ] A `rules` classifier with `extract` puts the extracted field into `labels.<name>`.

## Open questions

- Pipeline profiles live in `profiles/pipeline/`, and domain profiles (F41) in `profiles/domain/`. Should a domain profile select a pipeline profile by name, or override individual stages? Proposed: select by name, with optional knob overrides. F41 decides.
