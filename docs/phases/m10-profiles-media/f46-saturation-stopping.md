# F46 — Saturation stopping and "claims circulating online"

**Milestone:** M10 Domain profiles and media · **Status:** ⬜ Not started · **Depends on:** [F25](../m06-async-research/f25-fan-out-budget.md), [F43](f43-science-health.md) · **Unblocks:** —

[← Back to plan](../PLAN.md)

## Goal

Thorough runs should stop when more searching no longer changes the answer, not after a fixed number of sources. This feature adds saturation stopping: a run stops once two search rounds change no confidence label. It also adds a "claims circulating online" view, which compares popular claims with what the evidence actually says.

## Scope

**In scope**

- Round-based gathering for thorough depth, with a per-round snapshot of confidence labels.
- The saturation rule: stop when two consecutive rounds change no label.
- Interaction with the budget counselor: stop at saturation or at the budget cap, whichever comes first, and record which one stopped the run.
- Run page and report notes showing why the run stopped.
- The "claims circulating online" view and report table: popular claims vs. the evidence.

**Out of scope**

- Stopping rules for quick and standard depth: [F12](../m02-local-knowledge/f12-quick-depth-e2e.md) and [F25](../m06-async-research/f25-fan-out-budget.md).
- Confidence label calculation itself: [F23](../m05-source-quality/f23-claim-confidence.md), [F37](../m09-review/f37-reliability-judges-labels.md).

## Design notes

**Stopping rules by depth** (Design: Depth presets)

| Quick | Standard | Thorough |
| --- | --- | --- |
| First well-supported answer | Fixed sources per sub-question | Saturation: stop when two rounds change no confidence label |

- The budget counselor enforces the caps. Thorough has a large budget cap and asks before exceeding it. (Design: Depth presets)
- Most of the quality gain comes from verification, citation chasing and the stopping rule, not from longer thinking. (Design: Depth presets)
- The Director role decides when to stop. (Design: Model routing › Roles)
- Calibrated labels are high, moderate, low, contested and unverified. (Design: Source quality › Claim confidence)
- Coffee example, step 6: thorough depth keeps searching until new sources stop moving any outcome's certainty. (Design: Worked examples)

**Claims circulating online** (Design: Worked examples › Coffee, step 7): the report includes a "claims circulating online" table comparing popular claims with the evidence. In the coffee example, blogs, supplement sellers and "coffee cures X" pages are fetched but classified as promotional or tertiary and never count as support. These pages are where the circulating claims come from.

## Implementation sketch

These are suggestions, not decisions.

- Define a round as one pass of searches across all open sub-questions. After each round, snapshot `{claim_id: label}` for claims in scope.
- Saturation counter: if a round changes no label (and adds no new claim above a low-weight threshold), increment the counter; otherwise reset it. Stop at 2. Make the count a profile/depth knob.
- The judgment is mechanical, but let the Director add a reason line and catch edge cases (for example, a sub-question with zero claims shouldn't count as "saturated").
- Per sub-question saturation: a saturated sub-question stops early while others continue. The run ends when all are saturated or the budget runs out.
- Record `stop_reason` (`saturated`, `budget`, `retry_limit`, `user`) and the per-round label diffs on the run, and show them on the Run page.
- Circulating claims: collect claims whose only support is from promotional, tertiary or derived sources (or that appear widely but have low confidence). Pair each with the closest evidence-backed ledger claim. Render a table: popular claim, what the evidence says, confidence, citations.

## Acceptance criteria

- [ ] A run stops once two search rounds change no certainty label
- [ ] When the budget cap is hit before saturation, the run stops (or asks first, per thorough settings) and records `budget` as the stop reason.
- [ ] The Run page shows per-round label changes and the final stop reason.
- [ ] A sub-question with no supporting claims is never treated as saturated on its own.
- [ ] The coffee example's report includes a "claims circulating online" table pairing at least one promotional claim with the evidence-backed claim.

## Open questions

- Should "no change" also require no new claims, or only no label changes? The design says labels only; the sketch adds a guard.
- Is "claims circulating online" a health-only template element, or available to every profile? It is described only in the coffee example.
