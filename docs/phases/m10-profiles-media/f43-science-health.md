# F43 — Science base and health overlay

**Milestone:** M10 Domain profiles and media · **Status:** ⬜ Not started · **Depends on:** [F37](../m09-review/f37-reliability-judges-labels.md), [F41](f41-profile-system.md) · **Unblocks:** [F46](f46-saturation-stopping.md)

[← Back to plan](../PLAN.md)

## Goal

This ships the first real domain profiles: Science, for any empirical question, and the Health overlay on top of it. Claims become structured (population, exposure, outcome, design, effect size), so findings from different populations never merge. The coffee example is the test case. It should produce a per-outcome certainty table that a careful reader would trust.

## Scope

**In scope**

- `science.toml`: evidence pyramid, structured claim schema, replication counts, mechanism tier.
- `health.toml` (`extends = "science"`): PICO claims, GRADE-style certainty, trial-registry check, health disclaimer.
- Structured claim extraction against the profile's schema, with merge rules that respect it.
- Health reviewer checklist.
- Summary-of-findings output template.
- PubMed baseline and PubMed Central corpora, installed through the corpus manager.
- ClinicalTrials.gov check for trials whose reported outcomes differ from their registered ones.

**Out of scope**

- Climate and Software overlays: [F47](f47-remaining-profiles.md).
- Saturation stopping and the "claims circulating online" table: [F46](f46-saturation-stopping.md).
- Profile selection: [F42](f42-profile-auto-selection.md).

## Design notes

**Profile fields, science column** (Design: Domain profiles › What a profile defines)

- Evidence hierarchy: umbrella review > meta-analysis > RCT > cohort > case report > mechanism > opinion.
- Primary means the study's own data.
- Claim schema: population, exposure, outcome, design, effect size.
- Claim-kind weights: measurement > mechanism > opinion. Incentive: flag only.
- Evidence decay: slow (years).
- Preferred corpora: PubMed baseline, PubMed Central, OpenAlex, arXiv.
- Reviewer checklist: confounding, reverse causation, dose, extrapolation.
- Output template: summary-of-findings table.

**Built-in rows** (Design: Built-in profiles)

- Science: any empirical question; primary sources are peer-reviewed studies and datasets. Distinctive rules: evidence pyramid, structured claims, replication counts.
- Health overlay: medicine, nutrition, fitness; primary sources are trials, cohort data and clinical guidelines. Distinctive rules: PICO claims, GRADE-style certainty, trial-registry checks, mechanism tier, health disclaimer.

**Coffee and health worked example** (Design: Worked examples)

1. Scope: the classifier picks the health overlay. The system asks which outcomes (mortality, heart, cancer, sleep, pregnancy, anxiety), which population (healthy adults, pregnant people, people with hypertension) and what the exposure means (cups per day, caffeinated or decaf, filtered or unfiltered).
2. Plan: one sub-question per outcome, working down the evidence pyramid. Umbrella reviews and meta-analyses come first, then guideline bodies such as EFSA's caffeine safety opinion. Individual studies are used only where reviews disagree or are out of date.
3. Gather and weed: the local PubMed baseline and PMC subset answer most of it. Blogs, supplement sellers and "coffee cures X" pages are classified as promotional or tertiary and never count as support. Mouse and cell studies go to the mechanism tier.
4. Extract: claims carry population, exposure, outcome, design and effect size. A finding in older Japanese adults never merges into "coffee lowers mortality."
5. Review: the checklist covers confounding (coffee drinkers once smoked more), reverse causation (people who get sick often quit coffee), dose-response shape, relative vs. absolute risk, animal-to-human extrapolation, and whether Mendelian randomization agrees with observational studies. ClinicalTrials.gov catches outcome switching.
6. Report: a summary-of-findings table (outcome, direction, number of studies, certainty), a section on populations to be careful with, and the health disclaimer.

**Reviewer method checks** (Design: Adversarial review › checklist): design, sample size, confounders, multiple comparisons, generalizing beyond the study population.

## Implementation sketch

These are suggestions, not decisions.

- Claim schema → JSON schema for the Extractor role. Missing fields stay null and are shown as "unspecified," never guessed.
- Merge rule: claims merge only if population, exposure and outcome match (after normalization). Design and effect size attach as evidence on the merged claim.
- Map evidence levels from F37's evidence-level judge onto the hierarchy. GRADE-style certainty starts from design (RCT high, observational low), then moves down for risk of bias, inconsistency, indirectness and imprecision, or up for large effect and dose-response. Map the result onto the calibrated labels: high, moderate, low, contested, unverified.
- Replication count = number of independent origins (F23) supporting the same structured claim.
- Trial-registry check: look up NCT ids in passages, fetch the registry record through the fetch proxy (prefer the ClinicalTrials.gov API), and compare registered outcomes with reported ones.
- Corpora: add PubMed baseline and PMC entries to the corpus manager (F10) as background downloads. Check sizes and update schedules at install time; don't hard-code them.
- Output template `summary_of_findings`: rows per outcome, with columns outcome, direction, number of studies, certainty and key citations.
- Disclaimer text is a profile `notices` entry that the report renderer always includes.

## Acceptance criteria

- [ ] Coffee example produces a per-outcome certainty table
- [ ] A health claim for one population never merges with the same outcome in a different population. Covered by a test with two fixture studies.
- [ ] Mouse or cell-study passages are tagged at the mechanism tier and don't count toward human-outcome certainty.
- [ ] A fixture trial with outcome switching against its registry entry is flagged.
- [ ] Every health-profile report ends with the health disclaimer.
- [ ] PubMed baseline and PMC can be installed from Settings and searched through the hub.

## Open questions

- The PubMed baseline and PMC are named here but missing from the "Local corpora" table in the design (Being a good netizen). Confirm their install size and whether to ship a subset.
- How literally should GRADE be followed? The design says "GRADE-style," which leaves room for a simplified version.
- The design's coffee report also includes a "claims circulating online" table, but that view belongs to [F46](f46-saturation-stopping.md). The coffee acceptance test here covers only the certainty table.
