# F40 — Verdict card and accept flow

**Milestone:** M9 Source depth and adversarial review · **Status:** ⬜ Not started · **Depends on:** [F27](../m06-async-research/f27-watches.md), [F28](../m07-idea-clouds-export/f28-claim-graph-clustering.md), [F35](../m08-project-chat/f35-output-staleness-derived.md), [F38](f38-review-job.md) · **Unblocks:** —

[← Back to plan](../PLAN.md)

## Goal

Review results become a verdict card that users can accept or dismiss. Accepting one updates the claim ledger, adds contradicts edges to the idea map, and marks dependent reports and outputs stale. Claims and sources show when they were last reviewed. Reviews can also run automatically at the end of research runs and on a schedule, so the project's findings stay current without manual effort.

## Scope

**In scope**
- The Review screen: a verdict card for a passage, claim, source or report.
- Accept and dismiss actions, attributed to the user and recorded in the activity feed.
- On accept: apply the proposed confidence change, add contradicts edges, and mark dependent reports and outputs stale.
- "Last reviewed" dates on every claim and source.
- Optional auto-review of the top load-bearing claims at the end of each research run (a project setting).
- Status-check watches: re-run retraction and correction checks on a project's key sources, and notify only on change.
- Inbox entries and push notifications for finished reviews and watch changes.

**Out of scope**
- Running the review: [F38](f38-review-job.md), [F39](f39-debate-mode.md).
- The staleness mechanism itself: [F35](../m08-project-chat/f35-output-staleness-derived.md). This feature triggers it.
- The watch scheduler: [F27](../m06-async-research/f27-watches.md). This feature adds a new watch kind.
- Graph edges and the idea map: [F28](../m07-idea-clouds-export/f28-claim-graph-clustering.md), [F29](../m07-idea-clouds-export/f29-idea-map-ui.md).

## Design notes

**The verdict card** (Design: Adversarial review › The verdict card)

| Verdict | Meaning |
| --- | --- |
| Holds up | Survived every check; nothing material found |
| Holds with caveats | True as far as it goes, but narrower than stated or missing context |
| Weakened | Significant counter-evidence or method problems; confidence should drop |
| Contradicted | Better evidence says otherwise |
| Retracted or superseded | The source itself was withdrawn or replaced |

Under the verdict: a short summary, each finding with its severity and cited passages, and the proposed change to the claim's confidence. **Nothing changes until you accept it.** Accepting updates the ledger, adds contradicts edges to the idea map, and marks dependent reports and outputs stale.

**Keeping reviews current** (Design: Adversarial review)
- Every claim and source shows a "last reviewed" date.
- Optional auto-review of the top load-bearing claims at the end of each research run.
- Watches can re-run status checks (retractions, corrections) on a project's key sources and notify only on change.

**Screens** (Design: Web interface): Review is the verdict card for a passage, claim, source or report, showing findings, evidence and the proposed confidence change, with accept or dismiss. The Inbox shows finished reviews and watch updates.

**Permissions and attribution** (Design: Teams and permissions): editors start reviews. Every accepted verdict records who accepted it and appears in the project activity feed. Viewers can read reviews but can't accept them.

**Outputs** (Design: Project chat › Outputs): outputs are saved with the claim ids they were built from, and are marked stale when those claims change.

## Implementation sketch

Suggestions, not decisions.

1. Add `status` (`pending` | `accepted` | `dismissed`), `decided_by` and `decided_at` columns to `reviews`. Add `last_reviewed_at` to claims and sources, updated when any review of them completes, whatever the decision.
2. Review page (askama + htmx): verdict badge, summary, findings grouped by severity with tappable passage citations (reusing the F16 passage sheet), the proposed confidence change shown as a before → after label, and Accept / Dismiss buttons for editors. For report reviews, show a per-claim list of child verdicts with accept-all and per-claim accept.
3. Accept, in one transaction:
   - Set the claim's confidence label (or an override with a reason pointing to the review).
   - For each contradicting passage, create a `contradicts` edge.
   - Call F35's `mark_stale(claim_ids)` so reports and outputs that cite those claims get stale markers.
   - Write an activity-feed entry.
4. "Retracted or superseded" on a source review marks the source and caps its reliability, consistent with F22.
5. Auto-review: add a project setting `auto_review_load_bearing` (off by default). At run completion, enqueue an F38 review over the run's top load-bearing claims at the run's depth. Results land as pending cards in the Inbox.
6. Status-check watch: a new F27 watch kind that, on schedule, re-runs only F38 checklist step 1 (status check) on sources the project marks as key. Compare with the previous result and notify only when something changed.
7. Last-reviewed badges on claim rows, source rows and the claim detail view.

## Acceptance criteria

- [ ] Accepting a "Weakened" verdict lowers the claim's confidence and marks its report stale
- [ ] Dismissing a verdict changes nothing in the ledger and records who dismissed it.
- [ ] Accepting a "Contradicted" verdict adds a contradicts edge that appears in the idea map's contradiction view.
- [ ] A viewer sees the verdict card but no Accept or Dismiss buttons.
- [ ] A status-check watch on an unchanged source sends no notification. After a planted retraction, the next run notifies.
- [ ] With auto-review on, a finished run produces pending review cards in the Inbox for its top load-bearing claims.

## Open questions

- The design doesn't say how a project's "key sources" are picked for status-check watches. Suggestion: sources cited by load-bearing claims, plus any the user pins.
- When a confidence change from a review conflicts with a later automatic recomputation from F23 or F37, which wins? Suggestion: an accepted review acts like a user edit to the ledger, which runs respect, until a newer review or the user clears it.
- Auto-review cost needs to count against the run's budget or its own. The design says reviews have their own budget, so give auto-review a separate cap. See the budget-caps gap in [PLAN.md](../PLAN.md).
