# F38 — Review job and reviewer checklist

**Milestone:** M9 Source depth and adversarial review · **Status:** ⬜ Not started · **Depends on:** [F14](../m03-ledger-verification/f14-verification-stage.md), [F17](../m04-polite-web/f17-fetch-proxy.md), [F22](../m05-source-quality/f22-crossref-openalex.md) · **Unblocks:** [F39](f39-debate-mode.md), [F40](f40-verdict-card.md)

[← Back to plan](../PLAN.md)

## Goal

Users can select a passage, claim, source or report and ask for an adversarial review. A fresh reviewer agent tries to break the target and returns a structured verdict, much like a code-review agent run against a codebase. This is how users stress-test findings before relying on them.

## Scope

**In scope**
- Review targets: a highlighted passage in a report, source or note; a single ledger claim; a whole source; a report or output.
- A report or output review fans out to its load-bearing claims, and the user confirms the claim list first.
- A reviewer checklist loaded from TOML, with all seven checks.
- A fresh reviewer agent with no access to the writer's context, preferring a different model than the writer.
- A background job with its own budget. Fetches go through the polite proxy.
- A structured verdict result (verdict, summary, findings with severity and cited passages, proposed confidence change), stored with the job.
- "Review" entry points: select text on the Report page, plus buttons on claims and sources.

**Out of scope**
- Debate mode: [F39](f39-debate-mode.md).
- The verdict card UI, the accept flow, ledger updates, staleness, last-reviewed badges, auto-review and status-check watches: [F40](f40-verdict-card.md). F38 can show a plain read-only result page until then.
- Per-profile reviewer checklists such as health or products: [F43](../m10-profiles-media/f43-science-health.md), [F44](../m10-profiles-media/f44-products-profile.md).
- The general profile loader and `extends`: [F41](../m10-profiles-media/f41-profile-system.md).

## Design notes

**What can be reviewed** (Design: Adversarial review)
- A highlighted passage in a report, a source or your own notes.
- A single claim from the ledger.
- A whole source ("is this paper still sound?").
- A report or output, which fans out to its load-bearing claims: the ones its conclusion depends on.

**The reviewer's checklist** (a TOML profile, like the review skills in coding tools)
1. **Status check:** retractions, corrections, expressions of concern and errata (Crossref); a preprint whose published version changed the results; laws, standards or guidelines that were later superseded.
2. **Trace to primary:** does the original actually say this? Look for quote drift, misread numbers, stripped context, and a population or timeframe that quietly changed.
3. **Counter-evidence search:** deliberately search for refutations, failed replications, published comments and letters, and post-publication critique sites such as PubPeer.
4. **Independence audit:** are the supporting sources really independent, or one origin repeated?
5. **Method critique** (studies): design, sample size, confounders, multiple comparisons, and generalizing beyond the study population.
6. **Reasoning and framing:** correlation presented as causation, relative risk without absolute risk, cherry-picked timeframes, claims broader than the evidence.
7. **Currency:** is it still true, or has newer evidence moved past it?

**How it runs**
- The reviewer is a fresh agent that never saw the writing, so the work isn't grading itself. A different model from the writer is the recommended default.
- It's a background job with its own budget. Fetches go through the same polite proxy.
- A single claim runs without a plan checkpoint. A whole report shows its list of claims to review first.

**Verdicts** (Design: Adversarial review › The verdict card): holds up · holds with caveats · weakened · contradicted · retracted or superseded. Under the verdict: a short summary, each finding with its severity and cited passages, and the proposed change to the claim's confidence. Nothing changes until the user accepts (F40).

**Model routing** (Design: Model routing): the reviewer role is a large model, ideally from a different family than the writer. No self-grading: the reviewer avoids the writer's model when another is configured. Reasoning budget is high for the reviewer at thorough depth.

**Depth** (Design: Research pipeline › Depth presets): adversarial review is off at quick, covers top load-bearing claims at standard, and covers all load-bearing claims in debate mode at thorough.

## Implementation sketch

Suggestions, not decisions.

1. Add a `reviews` table (id, project_id, target_kind, target_id, span, depth, status, verdict, summary, proposed_confidence, reviewer_model, budget, cost, created_by), plus `review_findings` (review_id, check, severity, text) and `review_finding_passages`.
2. Add a `ReviewJob` job kind on the F6 queue. The hub builds a target bundle: claim text, cited passages and source metadata. The bundle excludes the writer's draft history and prompts.
3. Checklist: `profiles/review/general.toml` with `[[checks]]` entries (id, title, instructions, applies_to). Load and validate it at startup with a small serde struct.
4. Run each check as a step. Status check calls Crossref and OpenAlex from F22. Counter-evidence search uses SearXNG and the fetch proxy, and new pages are read, archived and extracted into passages. Independence audit reuses F23.
5. The reviewer model returns structured JSON. Validate that every finding cites passage ids that exist. Reuse F14's verifier to check those citations.
6. Load-bearing claims: a director-role call over the report picks the claims its conclusion depends on. The user confirms the list before fan-out, which creates one child review per claim.
7. Pick the model through F8's routing with a "not the writer's model" constraint, falling back with a trace note if no other model is configured.
8. UI entry points: text selection on the Report page offers "Review". Claim and source pages get a Review button. Results show on a simple page and in the Inbox.

## Acceptance criteria

- [ ] A planted citation to a retracted paper returns "Retracted or superseded"
- [ ] A review of a single claim starts without a plan checkpoint. A review of a report first shows its load-bearing claims for confirmation.
- [ ] The reviewer's prompt and trace contain no writer draft or writer reasoning.
- [ ] When two models are configured, the reviewer uses a different one than the writer, and the trace records which.
- [ ] Every finding cites at least one existing passage id. Findings that cite none are rejected and retried.
- [ ] A review that would exceed its budget stops and reports cost so far.

## Open questions

- **Checklist as TOML before the profile system.** The design says the checklist is a TOML profile, but the profile system arrives later in [F41](../m10-profiles-media/f41-profile-system.md). F38 should ship one built-in `general` checklist file with its own minimal loader and schema validation, using the field names F41 will adopt. F41 then absorbs this loader and adds `extends`, so health and products checklists can override checks.
- The reviewer agent runs on the tool loop from [H5](../mh-pipeline-core/h05-tool-loop.md), with turn-end validation from [H9](../mh-pipeline-core/h09-turn-end-validation.md) and judges from [H10](../mh-pipeline-core/h10-judges.md).
- "Load-bearing" has no precise definition in the design. Start with a director judgment, and revisit once F28's graph edges can rank claims by how many conclusions depend on them.
- Reviewing a chart or output (mentioned in the design intro) depends on [F34](../m08-project-chat/f34-outputs.md). Here it's treated as reviewing the claims the output was built from.
