# F12 — Quick depth end to end

**Milestone:** M2 Local knowledge · **Status:** ⬜ Not started · **Depends on:** [F9](f09-passage-store.md), [F10](f10-kiwix-corpus-manager.md), [F11](f11-notes-ingestion.md), [H4](../mh-pipeline-core/h04-context-assembler.md), [H7](../mh-pipeline-core/h07-input-stages.md), [H8](../mh-pipeline-core/h08-counselors.md), [H9](../mh-pipeline-core/h09-turn-end-validation.md), [H10](../mh-pipeline-core/h10-judges.md) · **Unblocks:** [F13](../m03-ledger-verification/f13-claim-ledger.md), [F19](../m04-polite-web/f19-searxng.md), [F24](../m06-async-research/f24-planner-checkpoint.md), [F26](../m06-async-research/f26-datasets-analyst.md)

[← Back to plan](../PLAN.md)

## Goal
This is the first real research run. You type a question, Searchy retrieves passages from Wikipedia and your notes, and it writes a short answer in which every claim cites a passage id. It proves the whole path from question to cited answer at the cheapest depth, and it creates the run skeleton that later milestones extend.

## Scope
**In scope**
- The New question screen, with depth fixed or defaulted to Quick. The Standard and Thorough choices can appear but stay disabled.
- A run pipeline skeleton with the named stages Scope → Plan → Gather → Read and extract → Score → Synthesize → Verify. Most stages are pass-throughs or minimal for now.
- Quick depth knobs: no plan checkpoint, clarifying questions only if ambiguous, 1–3 sub-questions, local corpora only, stop at the first well-supported answer.
- The director writes 1–3 sub-questions. Gather runs hybrid retrieval per sub-question. The writer drafts an answer that cites passage ids.
- Run progress over SSE (from F7) and a basic result page that renders citations as passage links.
- A run record of the profile (General for now), the depth, and the models used.

**Out of scope**
- Claims and the ledger → [F13](../m03-ledger-verification/f13-claim-ledger.md).
- Verification and retry → [F14](../m03-ledger-verification/f14-verification-stage.md). Quick depth calls for verifying load-bearing claims only, but that arrives with F14.
- The full report page with tap-to-passage → [F16](../m03-ledger-verification/f16-report-page.md).
- Plan checkpoint and Standard/Thorough depth → [F24](../m06-async-research/f24-planner-checkpoint.md), [F25](../m06-async-research/f25-fan-out-budget.md).
- Web sources → [F19](../m04-polite-web/f19-searxng.md).
- Profile selection → [F41](../m10-profiles-media/f41-profile-system.md)/[F42](../m10-profiles-media/f42-profile-auto-selection.md).

## Design notes
**Pipeline** (Design: Research pipeline): a research run is the chat harness's turn pipeline stretched over minutes or hours.
1. Scope: classify the question, check it against the ledger, and ask a clarifying question if it's ambiguous.
2. Plan: sub-questions, corpora, budget. (At Quick depth there is no checkpoint.)
3. Gather: search local corpora first.
4. Read and extract.
5. Score.
6. Synthesize: the writer drafts from claims, citing passage ids, never raw URLs.
7. Verify.

**Quick preset** (Design: Research pipeline › Depth presets)

| Knob | Quick |
| --- | --- |
| Plan checkpoint | Off |
| Clarifying questions | Only if ambiguous |
| Sub-questions | 1–3 |
| Sources searched | Local corpora |
| Stopping rule | First well-supported answer |
| Citation chasing | Off |
| Lateral reading | Off |
| Reasoning budget | Low everywhere |
| Verification | Load-bearing claims only |
| Retry limit | 1 |
| Synthesis | One draft |
| Adversarial review | Off |
| Budget cap | Small |

Quick uses small models everywhere except the director (Design: Model routing › Assignment layers).

**Citations** (Design: Knowledge store): your material is labeled "your notes" in citations. "Your notes" means documents you uploaded in the Library ([F11](f11-notes-ingestion.md); D6) and shared to the project.

**Async** (Design: Web interface › Async behavior): all state lives in SQLite, so closing the tab changes nothing. On reconnect, the page re-renders from the database.

## Implementation sketch
Suggestions only.
- Use the `searchy-pipeline` crate. The `Stage` trait and `TurnContext` come from [H1](../mh-pipeline-core/h01-pipeline-core-types.md) and the stage runner from [H6](../mh-pipeline-core/h06-stage-runner-classifiers.md). F12 adds the research stages (scope, retrieve, synthesize) and a run-level context holding the question, depth, profile, a scope filter, and an accumulator for passages and drafts. Tracing comes from [H3](../mh-pipeline-core/h03-turn-store-tracer.md), input stages from [H7](../mh-pipeline-core/h07-input-stages.md), counselors from [H8](../mh-pipeline-core/h08-counselors.md), turn-end validation from [H9](../mh-pipeline-core/h09-turn-end-validation.md) and judge classifiers from [H10](../mh-pipeline-core/h10-judges.md).
- `DepthPreset` struct holding all the knobs from the table. Only `quick` is populated now, with values loaded from config so they can be tuned.
- Citation format in the writer prompt: `[p:<passage_id>]`. After generation, parse the markers and drop any id that wasn't in the retrieved set, so the model can't invent ids.
- Clarifying question: if the director flags the question as ambiguous, the run goes to `awaiting_clarification` and the Inbox shows it.
- The run executes as a job (F6). Progress events are `stage_started`, `passages_found`, `draft_ready` and `done`.

## Acceptance criteria
- [ ] Answer cites Wikipedia and your notes by passage
- [ ] Every citation in the answer resolves to a passage that was retrieved in this run. Unknown ids are stripped.
- [ ] Citations to your notes are visibly labeled "your notes".
- [ ] Closing the browser mid-run and reopening shows the run's current state, rendered from the database.
- [ ] The run record stores the depth, the profile and the model used per role.
- [ ] An ambiguous question produces a clarifying question instead of an answer.

## Open questions
- "Load-bearing claims only" verification for Quick depends on F14. Until then, Quick answers are unverified and should say so.
- Ledger lookup at the Scope stage needs F13. For now, the Scope stage only classifies the question and decides whether to clarify.
