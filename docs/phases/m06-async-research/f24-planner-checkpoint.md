# F24 — Planner and plan checkpoint

**Milestone:** M6 Async research · **Status:** ⬜ Not started · **Depends on:** [F7](../m01-deployable-shell/f07-progress-notifications.md), [F12](../m02-local-knowledge/f12-quick-depth-e2e.md), [H12](../mh-pipeline-core/h12-error-recovery-reasoning.md), [H13](../mh-pipeline-core/h13-compaction-tool-disclosure.md) · **Unblocks:** [F25](f25-fan-out-budget.md), [F27](f27-watches.md), [F28](../m07-idea-clouds-export/f28-claim-graph-clustering.md), [F32](../m08-project-chat/f32-answerability-handoff.md), [F41](../m10-profiles-media/f41-profile-system.md)

[← Back to plan](../PLAN.md)

## Goal

Add the one human checkpoint in a research run. The director model scopes the question and proposes a plan: sub-questions, sources and a budget. The run then pauses and notifies you. You approve, edit or reject the plan from your phone, and only then does the run fan out.

## Scope

**In scope**

- Scope stage: classify the question type, check the claim ledger, ask a clarifying question if the question is ambiguous.
- Plan stage: the director proposes sub-questions, corpora and source types, and a budget.
- The `awaiting_approval` run state, with a push notification and an Inbox entry.
- The Plan review screen: approve, edit or reject sub-questions and the source list. Override depth knobs.
- The New question screen: type or dictate, pick a depth, set a budget.
- Depth presets as named knob bundles (plan checkpoint, clarifying questions, sub-question counts).
- Chat context injection (PLAN.md decision D11). The planner, and each research step it launches, receive the relevant project chat thread briefs from [H13](../mh-pipeline-core/h13-compaction-tool-disclosure.md) as a hidden context block. Apart from that block, each research step starts with a fresh context.
- Clarifying questions and any counselor `AskUser` during the run become Inbox items (D9). Only the asking step pauses.

**Out of scope**

- Running sub-questions in parallel and enforcing the budget: [F25](f25-fan-out-budget.md).
- Profile selection and the profile dropdown in plan review: [F41](../m10-profiles-media/f41-profile-system.md), [F42](../m10-profiles-media/f42-profile-auto-selection.md).
- Scheduled reruns: [F27](f27-watches.md).

## Design notes

Pipeline steps owned here (Design: Research pipeline):

1. **Scope.** Input stages classify the question (factual, comparative, literature review, data analysis), check it against the project's claim ledger, and ask a clarifying question if it's ambiguous.
2. **Plan.** The planner proposes sub-questions, which corpora and source types to use, and a budget. The run pauses in `awaiting_approval` and notifies you.

Depth preset knobs relevant here (Design: Research pipeline › Depth presets):

| Knob | Quick | Standard | Thorough |
| --- | --- | --- | --- |
| Plan checkpoint | Off | On | On |
| Clarifying questions | Only if ambiguous | Only if ambiguous | Always confirm scope |
| Sub-questions | 1–3 | 3–6 | 5–12 |
| Reasoning budget | Low everywhere | Medium for planner and judge | High for planner, judge and reviewer |
| Budget cap | Small | Medium | Large; asks before exceeding |

- Depth is independent of the domain profile. Every knob can be overridden per question in plan review. Values are starting defaults to tune. (Design: Depth presets)
- Input stages: scope check, ledger lookup, clarifying question, plan checkpoint. (Design: harness stage mapping)
- Director role: scopes questions, writes plans, decides when to stop. Large cloud model. Model assignment can be overridden per question in plan review. (Design: Model routing)
- Screens (Design: Web interface):
  - New question: type or dictate, pick depth (quick, standard, thorough), confirm the domain profile, set a budget.
  - Plan review: approve, edit or reject the proposed sub-questions and source list before fan-out.
  - Inbox: plans awaiting approval are the first item.
- Web Push (over HTTPS) or ntfy (D3) notifies you when a plan needs approval. (Design: Async behavior)
- Chat feeds research (PLAN.md D11): chat threads are compacted, and their important points are kept as a running summary. That summary is injected into planner and research-agent context. Research steps otherwise start fresh. H13 produces the brief, and this feature injects it.
- Questions mid-run (PLAN.md D9): a counselor's `AskUser` becomes an Inbox item. The step that asked pauses until it's answered, and other branches continue.
- Editors can start runs. Viewers cannot. (Design: Teams › Roles)

## Implementation sketch

Suggestions only.

1. Run state machine: `scoping → clarifying? → planning → awaiting_approval → approved | rejected → gathering …`. Persist every transition in SQLite so a hub restart resumes cleanly.
2. `depth_presets` as a config file (TOML), loaded at startup. The per-question override is a JSON blob on the question row.
3. Chat context:
   - Pick the thread briefs to include. The default is:
     - The thread that started the question (via F32's handoff, if any).
     - Plus project threads whose brief matches the question by embedding similarity, top 3.
   - Add them as one hidden "Project discussion so far" block in the planner's context, with each brief labelled by thread title and date.
   - Store the chosen thread ids on the run, so research steps get the same block and the run stays reproducible.
   - Briefs are derived text: they guide planning, but they are never cited as sources.
4. Scope stage:
   - Classifier role for question type.
   - Hybrid search over the ledger for existing claims, to show "already known" in the plan.
   - The director decides whether to ask a clarifying question. It's asked through H8's `AskUser` and becomes an Inbox item (D9). The run waits in `clarifying` until it's answered.
5. Plan stage: the director returns structured JSON with `sub_questions[]`, `sources[]` (corpora and source types), `budget` and `knobs`. Validate against a schema and retry on failure.
6. Plan review screen: an htmx form with an editable list of sub-questions (add, remove, reorder), source checkboxes, a budget field, knob overrides, and Approve / Re-plan / Reject.
7. Push: on entering `awaiting_approval`, send a Web Push or ntfy notification (D3) whose link opens the plan review page directly.
8. Quick depth skips the checkpoint, so [F12](../m02-local-knowledge/f12-quick-depth-e2e.md) behavior is unchanged.

## Acceptance criteria

- [ ] Approve a plan from a phone push notification
- [ ] A standard-depth question pauses in `awaiting_approval` and shows in the Inbox.
- [ ] Editing a sub-question in plan review changes what the run gathers.
- [ ] An ambiguous question triggers a clarifying question; thorough depth always confirms scope.
- [ ] A hub restart while a run is in `awaiting_approval` keeps the plan and the state.
- [ ] Quick depth runs without a checkpoint.
- [ ] A run started from a chat thread includes that thread's brief in the planner request. The trace shows it as a hidden block, and the run records which thread briefs it used.
- [ ] A clarifying question appears as an Inbox item, and answering it resumes planning.

## Open questions

- Scope and plan stages are input stages on the runner from [H6](../mh-pipeline-core/h06-stage-runner-classifiers.md) and [H7](../mh-pipeline-core/h07-input-stages.md). Plan approval stays a run state of its own (`awaiting_approval`), because it gates the whole run. Clarifying questions use H8's `AskUser`, which becomes an Inbox item (resolved by D9). Planner reasoning budgets come from [H12](../mh-pipeline-core/h12-error-recovery-reasoning.md).
- How many thread briefs to inject, and by what relevance rule. The default above is a starting point to tune.
- Dictation: the design says "type or dictate". Browser speech input on the textarea may be enough; confirm no server-side speech-to-text is expected.
- The full depth-preset table spans several features. See the depth-preset gap item in [PLAN.md](../PLAN.md#design-items-with-no-feature-yet).
- Rejection behavior: does a rejected plan close the question, or allow re-planning with feedback?
