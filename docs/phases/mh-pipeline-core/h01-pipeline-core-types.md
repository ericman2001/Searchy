# H1 — Pipeline crate and core types

**Milestone:** MH Pipeline core · **Status:** ⬜ Not started · **From harness:** F1 · **Depends on:** [F1](../m01-deployable-shell/f01-cargo-workspace.md) · **Unblocks:** [H2](h02-provider-adapters.md), [H3](h03-turn-store-tracer.md), [H6](h06-stage-runner-classifiers.md)

[← Back to plan](../PLAN.md)

## Goal

Create the `searchy-pipeline` crate and the types every later stage, counselor and validator shares. This replaces the "harness core crates" the design expected to import. The pipeline is built here, in Searchy's own workspace. Nothing runs yet; this feature fixes the vocabulary so H2–H14 can be built and tested against fakes.

## Scope

**In scope**

- A `searchy-pipeline` library crate in the F1 workspace, with no dependency on `axum` or the hub.
- The `Stage` trait, `StageKind`, `Verdict` and `CounselorAction` types.
- `TurnContext`, generalized for both research-run steps and project-chat turns.
- Core records: `Message`, `TraceEvent`, `Labels`, `Budget`.
- Pipeline-profile TOML types (serde) and parsing of a sample profile. Validation logic is out of scope.
- Unit tests that build a `TurnContext` and parse the sample profile.

**Out of scope**

- Model calls → [H2](h02-provider-adapters.md).
- Persistence and the tracer → [H3](h03-turn-store-tracer.md).
- Running stages, `needs` / `when` and profile validation → [H6](h06-stage-runner-classifiers.md).
- Domain profiles (evidence hierarchies, claim schemas) → [F41](../m10-profiles-media/f41-profile-system.md).

## Design notes

- Research runs are "the chat harness's turn pipeline stretched over minutes or hours." Classifiers, input stages, counselors and turn-end validation map onto research uses. (Design: Research pipeline › How the chat harness stages map)
- The harness defines one shared interface for all three stages. (Harness: Stage 1 › Shared stage interface)

```rust
trait Stage {
    fn name(&self) -> &str;
    fn kind(&self) -> StageKind;          // Deterministic | Model
    async fn run(&self, ctx: &mut TurnContext) -> Verdict;
}

enum Verdict {
    Pass,
    Transform { note: String },          // stage edited ctx in place
    Annotate { text: String },           // hidden note for the model
    Warn { reason: String },             // shown to user, turn continues
    Block { reason: String },            // stop, explain to user
    Retry { feedback: String },          // turn-end only: send model back
}
```

- Counselor actions are `Allow`, `Deny { reason }`, `Inject { text }`, `AskUser { question }` and `Abort { reason }`. (Harness: Stage 2 › Counselor actions)
- Core types from the harness, adapted. (Harness: Core services › Core types)
  - The harness `Session` record is **dropped** (PLAN.md decision D8). Each turn belongs to either a run step (F5's runs) or a chat thread (F31). Searchy has no separate session table.
  - `Message { id, owner, role, content_original, content_sent, hidden }`. `owner` is a `TurnOwner`, either a run step or a chat thread. It replaces `session_id`.
  - `TurnContext { scope, user_msg, history, pending_notes, tool_calls, draft, budget }`. `scope` replaces `session`.
  - `TraceEvent { turn_id, stage, event, verdict, latency_ms, tokens }`
- Classifiers label the input once per turn. Their results are cached on `TurnContext.labels`. (Harness: Configuration › Classifiers are a separate layer)
- A pipeline profile is TOML listing classifiers, `[[input]]`, `[[counselor]]` and `[[turn_end]]` stages, and `[turn_end_policy]`. (Harness: Configuration › Example profile)
- A pipeline profile decides which stages run. A domain profile decides what counts as good evidence. They are different files. A domain profile may later choose or extend a pipeline profile. (Design: Domain profiles)

## Implementation sketch

Suggestions, not decisions:

1. Add `crates/searchy-pipeline`. Use `async-trait` or native async-fn-in-trait, whichever the toolchain supports cleanly.
2. Generalize `TurnContext` with a `TurnOwner` enum. The two variants are `RunStep { run_id, step_id }` and `ChatThread { thread_id }`. Replace the harness `session` with `scope`. `scope` holds the project id, the user id and the `TurnOwner`. Keep the other fields from the harness. Don't define a `Session` type (D8).
3. Searchy extends the harness's `Retry` variant to `Retry { feedback: String, spans: Vec<SpanRef> }`, where an empty `spans` means the whole draft (PLAN.md D10). Define `SpanRef { sentence_id, reason }` here, and see [H9](h09-turn-end-validation.md) for how it's used.
4. Make `Budget` a plain struct: `max_tokens`, `max_tool_calls`, `max_wall_ms` and `max_cost_usd`, plus their spent counterparts. F25 fills in the values per user and project.
5. Make `Labels` a map from classifier name to a label, a score and extracted fields. The extracted fields cover `extract = "expr"` (H11).
6. Profile types: `PipelineProfile { classifiers, input, counselor, turn_end, turn_end_policy, tools, context, cache, reasoning, recovery }`. Leave unknown-name checks to H6, but use `#[serde(deny_unknown_fields)]`.
7. Add `profiles/pipeline/default.toml` with a minimal profile, and a test that parses it.

## Acceptance criteria

- [ ] `cargo test -p searchy-pipeline` passes, and a sample pipeline-profile TOML parses.
- [ ] `TurnContext` can be built for both a research step and a chat turn, and the tests cover both.
- [ ] Every `Verdict` and `CounselorAction` variant round-trips through serde JSON, as the tracer needs.
- [ ] A misspelled top-level key in the sample profile fails to parse with a readable error.
- [ ] The crate compiles without `axum`, `sqlx` or any network crate.
- [ ] There is no `Session` type. Every `Message` and `TurnContext` carries a `TurnOwner` (a run step or a chat thread).

## Open questions

- How much research-run state (sub-question, depth, domain profile id) should live on `TurnContext`, and how much in a run-level struct it points to? The suggestion is a run-level struct, so that `TurnContext` stays small.
- Resolved (D8): `Session` is dropped. Turns belong to a run step or a chat thread through `TurnOwner`.
