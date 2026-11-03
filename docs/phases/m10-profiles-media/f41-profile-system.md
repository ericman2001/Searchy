# F41 — Profile system

**Milestone:** M10 Domain profiles and media · **Status:** ⬜ Not started · **Depends on:** [F24](../m06-async-research/f24-planner-checkpoint.md) · **Unblocks:** [F42](f42-profile-auto-selection.md), [F43](f43-science-health.md), [F44](f44-products-profile.md), [F47](f47-remaining-profiles.md)

[← Back to plan](../PLAN.md)

## Goal

What counts as good evidence depends on the domain. This feature moves the quality rules out of code and into swappable TOML profiles that can extend each other. Every run records which profile it used, so reports can be reproduced. It is the base that the science, products and other built-in profiles plug into.

## Scope

**In scope**

- The TOML schema for a profile, covering every field in the table below.
- `extends = "<parent>"` inheritance: an overlay overrides only the fields it sets.
- A profiles folder: built-ins ship with the image, and custom profiles load from a mounted folder.
- Startup validation. A bad profile fails loudly with file, field and reason.
- The "General" profile, which holds the defaults that earlier features hard-coded.
- The profile recorded on every run, including the resolved version or hash.
- Manual choice of profile: project default and per question, shown in plan review.

**Out of scope**

- Automatic profile choice, confidence and per-sub-question profiles: [F42](f42-profile-auto-selection.md).
- Science/health content: [F43](f43-science-health.md). Products content: [F44](f44-products-profile.md). The other built-ins: [F47](f47-remaining-profiles.md).
- Per-profile model role overrides beyond carrying the field. The routing layer is [F8](../m02-local-knowledge/f08-model-routing.md); this feature only supplies the profile layer to it.

## Design notes

Quality rules live in swappable TOML profiles, not in code. A profile can extend another. You can override the choice at any level. (Design: Domain profiles)

**What a profile defines** (Design: Domain profiles › What a profile defines)

| Field | Science example | Products example |
| --- | --- | --- |
| Evidence hierarchy | Umbrella review > meta-analysis > RCT > cohort > case report > mechanism > opinion | Lab test with published method > long-term owner reports > hands-on review > spec sheet > listicle |
| What "primary" means | The study's own data | Direct testing or months of ownership |
| Claim schema | Population, exposure, outcome, design, effect size | Product, criterion, test method, firmware or revision |
| Claim-kind weights | Measurement > mechanism > opinion | Measurement and observation > spec > opinion |
| Incentive weight | Flag only | Heavy discount on opinions |
| Evidence decay | Slow (years) | Fast (months) |
| Preferred corpora | PubMed baseline, PMC, OpenAlex, arXiv | Test-site archives, video transcripts, owner forums |
| Reviewer checklist | Confounding, reverse causation, dose, extrapolation | Sponsorship, single unit, test conditions, firmware |
| Output template | Summary-of-findings table | Weighted comparison matrix |

- Claim kinds are shared across all profiles: measurement, observation, spec, opinion, mechanism. The kind decides how much incentive bias matters.
- Overlays declare `extends = "science"` and override only what differs. (Design: Built-in profiles)
- Custom profiles are TOML files in a profiles folder, validated at startup like the chat harness's stage profiles. (Design: Auto-selection)
- Overrides work at three levels: project default, question and sub-question. The profile used is recorded on every run. (Design: Auto-selection)
- The confidence rule is tunable in the profile TOML. The starting rule: "high" needs at least two independent origins, one of them primary, and no unresolved contradiction from a source of similar reliability. (Design: Source quality › Claim confidence)
- The domain profile is one of the model-assignment layers. For example, the health overlay might raise the verifier to a mid-size model. (Design: Model routing › Assignment layers)
- Depth is independent of profile: a health question can run at any depth. (Design: Depth presets)
- "General" is for anything else: it judges each source individually and uses the defaults described in the design doc. (Design: Built-in profiles)

**Migrating interim defaults.** Earlier features use hard-coded or interim defaults: F12 quick depth, F23 confidence rule, F36/F37 tier and reliability weights, and F38's reviewer checklist. This feature moves all of them into `general.toml`. After that, code reads them only through the resolved profile.

**Reliability weights and caps** (PLAN.md decision D7): reliability signals are stored per source, per project ([F21](../m05-source-quality/f21-source-classifier-trust.md)), and the weight of each signal is a domain-profile setting. Before this feature, F21, F22 and F37 use built-in General defaults: weights for `source_type`, `domain_trust`, `citation_impact`, `lateral_reputation`, `evidence_level`, `specificity`, `recency` and `conflict_of_interest`, plus status caps (`retracted`, `concern`). This feature moves them into `[reliability.weights]` and `[reliability.caps]` in `general.toml`. Changing the profile on a project recomputes that project's reliability scores.

## Implementation sketch

These are suggestions, not decisions.

- Crate `searchy-profiles`: `Profile` (raw, all fields `Option`), `ResolvedProfile` (all fields filled), loader, resolver and validator. Use `serde` + `toml`.
- Resolution: walk the `extends` chain to `general`. Child fields replace parent fields. Lists replace whole; don't merge them, because merging makes it hard to reason about what an overlay actually does. Detect cycles.
- Schema sketch:

```toml
id = "health"
extends = "science"
name = "Health overlay"
description = "Medicine, nutrition, fitness"

[evidence]
hierarchy = ["umbrella_review", "meta_analysis", "rct", "cohort", "case_report", "mechanism", "opinion"]
primary_means = "The study's own data"
decay = "years"            # none | years | months | days

[claims]
schema = ["population", "exposure", "outcome", "design", "effect_size"]
kind_weights = { measurement = 1.0, observation = 0.9, mechanism = 0.5, spec = 0.5, opinion = 0.2 }
incentive = "flag"         # flag | discount_opinions | heavy

[reliability.weights]       # D7; General defaults shown, overlays override per signal
source_type = 1.0
domain_trust = 1.0
citation_impact = 0.5
lateral_reputation = 1.0
evidence_level = 1.0
specificity = 0.5
recency = 0.5
conflict_of_interest = 0.0  # flag only

[reliability.caps]
retracted = 0.05
concern = 0.4

[confidence.high]
min_independent_origins = 2
require_primary = true

corpora.preferred = ["pubmed_baseline", "pmc", "openalex", "arxiv"]
reviewer.checklist = ["confounding", "reverse_causation", "dose", "extrapolation"]
output.template = "summary_of_findings"
notices = ["health_disclaimer"]
[roles]                    # optional layer for F8
verifier = "mid"
```

- Validation checks: unknown reliability signal names in `[reliability.weights]`, weights and caps outside 0–1, unknown keys, unknown parents, cycles, hierarchy labels that are known evidence levels, claim kinds limited to the five shared kinds, known output template ids, known corpus ids (warn if a corpus isn't installed; don't fail).
- Store each run's `profile_id` plus a hash of the resolved profile. Also snapshot the resolved TOML, so reruns reproduce even after the file is edited.
- Plan review (F24) shows a profile dropdown. Changing it re-plans the run. F42 later adds auto-selection.
- Steps: schema and loader → resolver → validator with good errors → `general.toml` migration → run recording → dropdown in plan review and project settings.

## Acceptance criteria

- [ ] A custom overlay that changes one field loads and shows in plan review
- [ ] A broken profile (bad key, unknown parent or cycle) stops startup with a message naming the file and field.
- [ ] An overlay with `extends = "science"` inherits every field it doesn't set; a unit test compares the resolved output.
- [ ] Every run row stores the profile id and resolved-profile hash, and the report shows the profile name.
- [ ] Hard-coded defaults from F12/F23/F36–F38 are gone from code and live in `general.toml`. Behaviour is unchanged on the existing tests.
- [ ] The reliability weights and caps from F21, F22 and F37 are read from `[reliability.weights]` and `[reliability.caps]`. Switching a project to an overlay that changes one weight recomputes only that project's source scores.
- [ ] The profile can be set as a project default and overridden per question.

## Open questions

- Should list fields merge or replace on `extends`? The sketch says replace; confirm once real overlays exist.
- Where do custom profiles live in the container? Probably `/data/profiles`. Should they be editable from the Settings UI, or only as files?
- F41 lists F9 as a dependency, but it's unclear what it needs from F9. Most likely `preferred corpora` has to name indexed corpora. F24 is the real driver.
- Reuse the startup validation approach from the pipeline profiles in [H6](../mh-pipeline-core/h06-stage-runner-classifiers.md). Decide whether domain profiles and pipeline profiles live in one TOML file or two that reference each other.
