# F42 — Profile auto-selection

**Milestone:** M10 Domain profiles and media · **Status:** ⬜ Not started · **Depends on:** [F25](../m06-async-research/f25-fan-out-budget.md), [F41](f41-profile-system.md) · **Unblocks:** —

[← Back to plan](../PLAN.md)

## Goal

When you ask a question, Searchy picks the right domain profile on its own. It shows its choice with a confidence and a one-line reason, so you can see why. If it isn't sure, it asks you instead of guessing. Mixed questions are split, so each part is judged by the right rules.

## Scope

**In scope**

- A Scope-stage classifier that proposes a profile with a confidence and a one-line reason.
- A low-confidence path that asks the user instead of guessing.
- A plan-review dropdown that shows the choice. Changing it re-plans the run.
- Per-sub-question profiles for mixed questions.
- Precedence of overrides: project default, question and sub-question.

**Out of scope**

- The profile schema, loading and run recording: [F41](f41-profile-system.md).
- Content of specific profiles: [F43](f43-science-health.md), [F44](f44-products-profile.md), [F47](f47-remaining-profiles.md).
- Running sub-questions in parallel: [F25](../m06-async-research/f25-fan-out-budget.md). This feature only labels each sub-question.

## Design notes

**Auto-selection** (Design: Domain profiles › Auto-selection)

- At the Scope stage, a classifier proposes a profile with a confidence and a one-line reason, such as "health overlay: effect of a food on outcomes in people."
- Plan review shows the choice as a dropdown; changing it re-plans the run.
- Low confidence means the system asks instead of guessing.
- Mixed questions get a profile per sub-question. "Is coffee healthy, and which espresso machine should I buy?" splits into a health branch and a products branch.
- Overrides work at three levels: project default, question and sub-question. The profile used is recorded on every run so reports are reproducible.

**Related rules**

- Profile selection is a Classifier-role job: a small model, with rules first where possible. (Design: Model routing › Roles)
- Classifiers in the harness mapping include question type. Scope also classifies the question as factual, comparative, literature review or data analysis. (Design: Research pipeline)
- Clarifying questions follow the depth preset: quick and standard ask only if the question is ambiguous; thorough always confirms scope. (Design: Depth presets)
- Built-in profiles to choose from: Science (with Health, Climate and Software overlays), Products, News and events, Legal and policy, History and humanities, Technical docs, Finance and companies, General. (Design: Built-in profiles)

## Implementation sketch

These are suggestions, not decisions.

- Each profile carries a short `description` and optional `selection.keywords` / `selection.examples` in its TOML. The classifier prompt is built from these, so custom profiles are selectable without code changes.
- Two passes. First, cheap rules (keyword hits, project default). Second, a Classifier-role model call returning `{ profile, confidence, reason, split: [{ text, profile, confidence, reason }] }` as structured JSON.
- Thresholds live in config: for example `auto_accept >= 0.75`, `ask < 0.5`. In between, accept but highlight the choice in plan review.
- Precedence when resolving: sub-question override > question override > classifier pick > project default > `general`. If there's an explicit project default, decide whether the classifier still runs (see Open questions).
- Splitting: the planner (F24) receives the split hints and produces sub-questions tagged with their profile. Sub-questions with different profiles use different evidence rules and output templates. The report gets one section per branch.
- Record on the run: the classifier output, its confidence and reason, and any user override with who made it.
- Steps: profile descriptions → rules pass → classifier call → ask-on-low-confidence via the clarifying-question path → dropdowns per question and sub-question → re-plan on change → split support.

## Acceptance criteria

- [ ] "Is coffee healthy, and which espresso machine should I buy?" splits into health and products branches
- [ ] A clearly single-domain question ("What does GDPR Article 17 require?") gets one profile with a reason shown in plan review.
- [ ] A low-confidence classification produces a clarifying question instead of a silent pick.
- [ ] Changing the dropdown in plan review re-plans the run with the new profile.
- [ ] A sub-question override beats the question override, which beats the project default. Unit tests cover the precedence.
- [ ] The run record stores the classifier's proposal, confidence, reason and any override.

## Open questions

- Should an explicit project default skip the classifier, or should the classifier still run and only flag a mismatch?
- How should a mixed-profile run render its final report: one report with sections, or linked reports per branch?
- Per-sub-question profiles depend on F25's fan-out, which F42 doesn't list as a dependency. In practice F25 will already be done before this feature.
