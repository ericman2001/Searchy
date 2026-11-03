# F14 — Verification stage

**Milestone:** M3 Ledger and verification · **Status:** ⬜ Not started · **Depends on:** [F13](f13-claim-ledger.md), [H9](../mh-pipeline-core/h09-turn-end-validation.md), [H10](../mh-pipeline-core/h10-judges.md) · **Unblocks:** [F15](f15-number-cross-check.md), [F16](f16-report-page.md), [F31](../m08-project-chat/f31-project-chat-loop.md), [F38](../m09-review/f38-review-job.md)

[← Back to plan](../PLAN.md)

## Goal
Check every cited sentence in a report against the passage it cites, before the report ships. Unsupported sentences go back to the writer with reasons. If they still fail after the retry limit, they ship visibly marked as unverified. This turn-end stage is what makes "every claim links to a passage" trustworthy, not just decorative.

## Scope
**In scope**
- A citation-completeness check: every factual sentence must cite at least one passage id.
- A judge per citation that labels it supports, partially supports, or does not support.
- A sentence-level retry back to Synthesize (PLAN.md decision D10). Only the failing sentences are sent back, with their reasons, up to the depth's retry limit. Passing sentences are never rewritten.
- After the limit, unsupported sentences are rewritten, removed, or kept and marked unverified.
- Verifier routing through F8: small model first, escalating uncertain cases.
- Depth knobs for Quick and Standard: load-bearing claims only versus every citation.
- Verification results stored per sentence and citation, for the report page to show.

**Out of scope**
- The number re-read → [F15](f15-number-cross-check.md).
- The second verifier model from another family at Thorough depth → [F25](../m06-async-research/f25-fan-out-budget.md). This feature should make it pluggable.
- Contradiction disclosure in reports → [F23](../m05-source-quality/f23-claim-confidence.md)/[F28](../m07-idea-clouds-export/f28-claim-graph-clustering.md).
- The report UI → [F16](f16-report-page.md).

## Design notes
**Turn-end verification** (Design: Source quality and verification › Verification)
- Every sentence in a report that makes a factual claim must cite at least one passage id.
- A judge model checks each citation: does the passage support, partially support, or not support the sentence?
- Unsupported claims are rewritten or removed on retry. After the limit, they ship marked as unverified.

**Pipeline step 7** (Design: Research pipeline): failures go back to step 6 with the reasons, up to the retry limit, then ship with visible flags.

**Harness mapping** (Design: Research pipeline): turn-end validation covers claim-to-passage verification, citation completeness and contradiction disclosure.

**Depth knobs** (Design: Depth presets)

| Knob | Quick | Standard | Thorough |
| --- | --- | --- | --- |
| Verification | Load-bearing claims only | Every citation | Every citation, plus a second verifier model; disagreements flagged |
| Retry limit | 1 | 2 | 3 |

**Verifier role** (Design: Model routing): small model first, escalating unclear cases. Small models handle format checks and clear support or no-support cases well. They miss subtle mismatches, such as a different population or hedged wording, so the verifier escalates anything it's unsure of.

**Sentence-level retries** (PLAN.md D10): retries target only the failing sentences. This validator returns H9's `Retry { feedback, spans }`, with one `SpanRef` per failing sentence.

**Reuse** (Design: Project chat): the same verifier checks chat answers exactly as it checks reports ([F31](../m08-project-chat/f31-project-chat-loop.md)). Build it as a reusable component.

## Implementation sketch
Suggestions only.
- Module `verify` in `searchy-pipeline`, with `verify(draft, passages, policy) -> VerifyReport`, usable by both runs and chat. Register it as a turn-end validator in [H9](../mh-pipeline-core/h09-turn-end-validation.md). The per-citation verifier call uses the judge helper from [H10](../mh-pipeline-core/h10-judges.md).
- Steps:
  1. Split the draft into sentences.
  2. Classify each sentence as factual or not (a cheap classifier, or the verifier itself).
  3. For factual sentences, check that at least one citation is present.
  4. For each (sentence, passage) pair, call the verifier, which returns `{verdict: supports|partial|unsupported, uncertain: bool, reason}`.
  5. If `uncertain`, re-ask the escalation model.
- "Load-bearing" for Quick: sentences in the conclusion or summary, or sentences the director marks as load-bearing in its outline. Store the rule in the depth preset.
- Retry: use the span-retry loop from [H9](../mh-pipeline-core/h09-turn-end-validation.md). Sentence splitting comes from H9's segmented draft rather than being reimplemented here. The validator returns `Retry { feedback, spans }`, with one `SpanRef` per failing sentence carrying its verdict and reason. The writer rewrites or drops only those sentences. H9 splices them back in and re-runs this validator on the replaced sentences only. When the limit is reached, H9's per-sentence `Warn` maps to the "unverified" status on just those sentences.
- Table: `verifications(run_id, sentence_idx, passage_id, verdict, model, escalated, reason, attempt)`, plus a `status` per sentence (`verified|partial|unverified|uncited`).

## Acceptance criteria
- [ ] Planted unsupported sentence is caught and removed or flagged
- [ ] A factual sentence with no citation fails the completeness check and is sent back on retry.
- [ ] A citation where the passage covers a different population is labeled partial or unsupported, through escalation if the small model is unsure.
- [ ] The number of retries never exceeds the depth's retry limit. After the limit, failing sentences carry an "unverified" status in the stored report.
- [ ] Every verifier call is recorded with its model and whether it escalated.
- [ ] The same `verify` function can be called on a standalone answer outside a run (a unit test is enough here).
- [ ] On retry, only the failing sentences are rewritten. The other sentences, and their stored verification rows, are unchanged.

## Open questions
- How does the verifier signal `uncertain`? Coordinate with [F8](../m02-local-knowledge/f08-model-routing.md).
- Classifying sentences as factual or not is a judgment call. Too strict means flagging transitions; too loose means letting claims through uncited.
