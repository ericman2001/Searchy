# F39 — Debate mode

**Milestone:** M9 Source depth and adversarial review · **Status:** ⬜ Not started · **Depends on:** [F38](f38-review-job.md) · **Unblocks:** —

[← Back to plan](../PLAN.md)

## Goal

At thorough depth, a review becomes a structured debate. A prosecutor builds the strongest case against the claim, a defender answers it, and a judge writes the verdict from both sides, citing passages. This surfaces stronger counter-arguments than a single reviewer would, and also guards against a reviewer that is too harsh.

## Scope

**In scope**
- A debate variant of the F38 review job, selected automatically at thorough depth, with an optional per-review override.
- Three roles: prosecutor, defender and judge, each a separate agent call with its own context.
- Prosecutor and defender both must cite passages. They may trigger new fetches for counter-evidence or supporting evidence through the proxy.
- A judge that reads both briefs and writes the standard verdict (same schema as F38), citing passages from both sides.
- Storing and displaying both briefs alongside the verdict.

**Out of scope**
- Verdict card and accept flow: [F40](f40-verdict-card.md).
- The base reviewer checklist and job plumbing: [F38](f38-review-job.md).

## Design notes

**Debate** (Design: Adversarial review › How it runs): "The `thorough` depth uses a debate: a prosecutor builds the strongest case against the claim, a defender answers, and a judge writes the verdict from both, citing passages."

**Depth** (Design: Research pipeline › Depth presets): at thorough, adversarial review covers all load-bearing claims in debate mode. Reasoning budget is high for the planner, judge and reviewer.

**Independence from the writer** (Design: Adversarial review › How it runs): reviewers are fresh agents that never saw the writing. A different model from the writer is the recommended default.

**Model routing** (Design: Model routing): the reviewer role is a large model, ideally from a different family than the writer. No self-grading.

**Verdict schema** (Design: Adversarial review › The verdict card): holds up · holds with caveats · weakened · contradicted · retracted or superseded, plus a summary, findings with severity and cited passages, and a proposed confidence change.

## Implementation sketch

Suggestions, not decisions.

1. Add a `mode` column to `reviews` (`single` | `debate`), and a `review_briefs` table (review_id, side: prosecutor/defender, text, cited passage ids, model).
2. Prosecutor step: run F38's checklist in an adversarial framing and produce a brief with cited passages. It may run counter-evidence searches.
3. Defender step: receives the prosecutor brief and the claim's supporting passages, answers each point with citations, and may search for supporting evidence.
4. Judge step: receives the claim, both briefs and the cited passages, but not the writer's draft. It outputs the F38 verdict JSON. Its citations must include at least one passage from each brief.
5. Model choice: if three or more models are configured, use different ones for prosecutor, defender and judge. Always avoid the writer's model for the judge. Record the actual models in the trace.
6. Budget: split the review budget across the three roles. If the budget runs out mid-debate, the judge rules on what exists and the summary notes it.
7. UI: the review result page shows the verdict, then collapsible prosecutor and defender briefs with their citations.

## Acceptance criteria

- [ ] Both sides cite passages and the verdict cites both
- [ ] A review started at thorough depth runs in debate mode by default. One started at standard runs as a single review.
- [ ] The judge's trace shows it received both briefs and no writer draft.
- [ ] A brief with no passage citations is rejected and retried, within the retry limit.
- [ ] When the budget runs out after the prosecutor step, the job still produces a verdict and flags it as incomplete.

## Open questions

- The design doesn't specify rounds: is it one exchange, or does the prosecutor get a rebuttal? Start with one exchange (prosecution, defense, judgment) and make the round count a config knob.
- Whether the judge uses the "judge" role from the depth presets or the "reviewer" role is unclear. Suggestion: route it as the reviewer role with high reasoning.
- The prosecutor, defender and judge each run on the tool loop from [H5](../mh-pipeline-core/h05-tool-loop.md). The judge's structured verdict uses the judge helper from [H10](../mh-pipeline-core/h10-judges.md).
