# F25 — Sub-question fan-out and budget counselor

**Milestone:** M6 Async research · **Status:** ⬜ Not started · **Depends on:** [F24](f24-planner-checkpoint.md), [H8](../mh-pipeline-core/h08-counselors.md), [H12](../mh-pipeline-core/h12-error-recovery-reasoning.md) · **Unblocks:** [F42](../m10-profiles-media/f42-profile-auto-selection.md), [F46](../m10-profiles-media/f46-saturation-stopping.md)

[← Back to plan](../PLAN.md)

## Goal

Turn an approved plan into parallel work that stays within budget. Each sub-question becomes its own job on a worker. A budget counselor tracks spending as the run goes, stops or asks before any cap is exceeded, and the run reports what it cost.

## Scope

**In scope**

- Fan-out: one job per approved sub-question, run in parallel on workers.
- Joining results back into one run for scoring, synthesis and verification.
- Budget counselor: track cost per role, per run, per project and per user. Enforce caps.
- Asking before exceeding a cap, through the Inbox and a push notification.
- Cost shown live on the Run screen and in the final report.
- Depth knobs that apply during gather: stopping rule (first answer, fixed sources), citation chasing hops, synthesis passes, retry limit.
- Loop guard counselor.

**Out of scope**

- Saturation stopping for thorough runs: [F46](../m10-profiles-media/f46-saturation-stopping.md).
- Citation chasing itself: [F36](../m09-review/f36-provenance-tiers.md).
- Adversarial review passes: [F38](../m09-review/f38-review-job.md), [F39](../m09-review/f39-debate-mode.md).
- Per-sub-question profiles: [F42](../m10-profiles-media/f42-profile-auto-selection.md).

## Design notes

- Gather: approved sub-questions fan out to workers. Each searches local corpora first, then the web through the fetch proxy. (Design: Research pipeline, step 3)
- Counselors include budget and loop guard. (Design: harness stage mapping)
- Depth presets: every knob can be overridden per question in plan review, and the budget counselor enforces the caps. (Design: Depth presets)

| Knob | Quick | Standard | Thorough |
| --- | --- | --- | --- |
| Stopping rule | First well-supported answer | Fixed sources per sub-question | Saturation (F46) |
| Synthesis | One draft | One draft and a self-critique pass | Two drafts, critiqued and merged |
| Retry limit | 1 | 2 | 3 |
| Budget cap | Small | Medium | Large; asks before exceeding |

- Budgets and keys (Design: Teams and permissions):
  - Cloud API keys are set once by the admin; spending is tracked per user and per project.
  - Budget caps apply at three levels: instance, user and project. A run that would exceed any of them asks first.
- The job queue schedules fairly across users, so one person's thorough run can't starve everyone else. (Design: Teams › Attribution and coordination)
- Per-role costs feed the budget counselor, so you can see where money goes and which role to downsize. (Design: Model routing › Rules)
- Fallback chains record which model actually ran. (Design: Model routing › Rules)
- Run screen: live progress over SSE: current step, sources fetched, claims found, cost so far. (Design: Web interface)
- Reviews are background jobs with their own budget. (Design: Adversarial review)

## Implementation sketch

Suggestions only.

1. On approval, create a parent run job and one child job per sub-question. The parent waits on the children, then runs score, synthesize and verify.
2. Children run gather, read and extract. Results land in the shared store keyed by run and sub-question.
3. Cost ledger: `cost_events (run_id, job_id, user_id, project_id, role, model, tokens_in, tokens_out, usd, at)`, written by the model router from [F8](../m02-local-knowledge/f08-model-routing.md).
4. Budget counselor: extend H8's starter budget counselor on the `BeforeModelCall` hook ([H8](../mh-pipeline-core/h08-counselors.md)) with run, project, user and instance caps. It runs before each model call:
   - Estimate the call's cost.
   - Check the run cap, project cap, user cap and instance cap.
   - On a breach, pause the run in an `awaiting_budget` state and notify.
5. Caps: `budget_caps (scope, scope_id, period, usd)` where scope is instance, user or project. Edited by admins (instance, user) and owners (project).
6. Loop guard: configure H8's loop-guard counselor to also cap repeated identical searches per job.
7. Fair scheduling: round-robin across users when picking the next queued child job.
8. Run screen: stream cost by role over SSE. The report footer shows total cost and the model used per role.

## Acceptance criteria

- [ ] Thorough run stays within its budget and reports cost
- [ ] A five-sub-question plan runs as five parallel child jobs and joins into one report.
- [ ] A run that would exceed the project cap pauses and asks, and continues only after approval.
- [ ] Spending is visible per role, per run, per project and per user.
- [ ] With two users queueing runs, neither user's jobs wait for all of the other's to finish.
- [ ] A hub restart mid-fan-out resumes the remaining child jobs.

## Open questions

- This feature absorbs two gap items from [PLAN.md](../PLAN.md#design-items-with-no-feature-yet): three-level budget caps, and fair scheduling (unless F6 already took it). Confirm.
- Cap sizes for "small", "medium" and "large" are not defined in the design.
- Thorough depth also relies on [F45](../m10-profiles-media/f45-video-transcripts.md), [F46](../m10-profiles-media/f46-saturation-stopping.md) and academic sources. Until those exist, "thorough" here means the knobs that are implemented.
