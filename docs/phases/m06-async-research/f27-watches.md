# F27 — Watches

**Milestone:** M6 Async research · **Status:** ⬜ Not started · **Depends on:** [F24](f24-planner-checkpoint.md) · **Unblocks:** [F40](../m09-review/f40-verdict-card.md)

[← Back to plan](../PLAN.md)

## Goal

Let a question keep watch over time. A watch reruns a question on a schedule and notifies you only when something changed: new claims, or a confidence that moved. You stop rechecking topics by hand, and the Inbox stays quiet when there's nothing new.

## Scope

**In scope**

- Putting a question on a watch, with a schedule (for example weekly).
- Scheduled reruns that reuse the approved plan without a new checkpoint (unless the budget would be exceeded).
- Diffing the ledger before and after a rerun: new claims, changed confidence, new contradictions.
- Notifying (Inbox, plus Web Push or ntfy per D3) only when the diff isn't empty.
- A "watch update" view that shows only what's new.
- Pausing, editing and deleting watches.

**Out of scope**

- Status-check watches that re-run retraction checks on key sources: [F40](../m09-review/f40-verdict-card.md).
- Product watches for new models and firmware: [F44](../m10-profiles-media/f44-products-profile.md).
- Budget enforcement itself: [F25](f25-fan-out-budget.md).

## Design notes

- Project → questions → runs. A question can be rerun or put on a watch. (Design: Knowledge store › Organization)
- Inbox shows watch updates. (Design: Web interface › Screens)
- Runs add to the store instead of starting over, so reruns are cheap and reuse archived pages. (Design: Knowledge store; Being a good netizen › Archiving)
- Later uses of watches (Design: Adversarial review › Keeping reviews current; Worked examples › Robot vacuum):
  - Watches can re-run status checks (retractions, corrections) on a project's key sources and notify only on change.
  - A products watch flags new models, firmware changes and reliability reports.
- News profile has very fast decay; products decay in months. Decay later informs sensible watch schedules. (Design: Domain profiles)
- Every run records who did it, shown in the activity feed. A watch-triggered run should be attributed to the user who created the watch. (Design: Teams › Attribution)
- A run that would exceed a budget cap asks first. (Design: Teams › Budgets)

## Implementation sketch

Suggestions only.

1. Table: `watches (id, question_id, created_by, schedule, last_run_id, next_run_at, paused, notify_on)`.
2. Scheduler: a hub task that wakes every minute, finds due watches and enqueues a run. The schedule can be a simple interval or a cron expression.
3. Rerun: copy the last approved plan and skip `awaiting_approval`. Still pass through the budget counselor.
4. Diff: snapshot ledger state for the question's claims before the run (claim ids, confidence, support and contradiction counts). Compare after. Store the diff as `watch_updates (watch_id, run_id, new_claims, changed_claims, new_contradictions)`.
5. Notify only if the diff is non-empty. The push links to the watch update view.
6. Watch update view: lists only the new and changed claims, with their citations.
7. UI: a "Watch this question" control on the question page, with schedule options and pause or delete.

## Acceptance criteria

- [ ] Weekly watch notifies only on change
- [ ] A watch rerun with no new or changed claims produces no notification and no Inbox item.
- [ ] A rerun that adds a claim notifies once and the update view shows only that claim.
- [ ] Watch reruns skip the plan checkpoint but still respect budget caps.
- [ ] A paused watch does not run.
- [ ] Watches survive a hub restart and run at their next due time.

## Open questions

- Whether a rerun should re-plan when the ledger has changed a lot, or always reuse the original plan.
- How "changed" is defined before calibrated labels exist ([F37](../m09-review/f37-reliability-judges-labels.md)). Until then, use corroboration counts from [F23](../m05-source-quality/f23-claim-confidence.md).
- Rerun cost: archived pages are reused, but search results need fresh fetches to find anything new. Confirm the per-watch budget default.
