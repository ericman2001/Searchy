# F44 — Products profile

**Milestone:** M10 Domain profiles and media · **Status:** ⬜ Not started · **Depends on:** [F37](../m09-review/f37-reliability-judges-labels.md), [F41](f41-profile-system.md), [F45](f45-video-transcripts.md) · **Unblocks:** —

[← Back to plan](../PLAN.md)

## Goal

This adds the Products profile for buying and comparing things. Product research is shaped by money: affiliate links, sponsorships and listicles copying each other. This profile discounts sponsored opinions while keeping useful observations. It produces a weighted comparison matrix based on your own priorities, with evidence behind every cell.

## Scope

**In scope**

- `products.toml`: evidence hierarchy, claim schema, claim-kind weights, fast decay, reviewer checklist, comparison-matrix template.
- Claim kinds applied in extraction: measurement, observation, spec, opinion.
- An incentive counselor that detects affiliate links, sponsorship disclosures, free review units and listicle networks.
- Product entity resolution: regional names and revisions of the same model.
- Priority weighting from the scoping answers (for example "pet hair 3×, mopping 0×").
- Live volatile facts: prices and availability fetched at report time, never stored as claims.
- Within-tester comparison and a cross-tester agreement check.
- The weighted models × criteria matrix and a "who it's for" summary per model.

**Out of scope**

- Video transcript ingestion with timestamps: [F45](f45-video-transcripts.md). This profile uses it once it exists.
- Watches that flag new models and firmware changes: built on [F27](../m06-async-research/f27-watches.md); this feature only configures the watch.
- Profile selection: [F42](f42-profile-auto-selection.md).

## Design notes

**Profile fields, products column** (Design: Domain profiles › What a profile defines)

- Evidence hierarchy: lab test with published method > long-term owner reports > hands-on review > spec sheet > listicle.
- Primary means direct testing or months of ownership.
- Claim schema: product, criterion, test method, firmware or revision.
- Claim-kind weights: measurement and observation > spec > opinion.
- Incentive weight: heavy discount on opinions.
- Evidence decay: fast (months).
- Preferred corpora: test-site archives, video transcripts, owner forums.
- Reviewer checklist: sponsorship, single unit, test conditions, firmware.

Claim kinds are shared across profiles. The kind decides how much incentive bias matters, so a sponsored reviewer's footage still counts as an observation, while their "best of the year" verdict is discounted. (Design: Domain profiles)

**Built-in row:** Products is for buying and comparing things. Its primary sources are tests with published methods and ownership reports. Distinctive rules: incentive detection, product entity resolution, priority weighting, fast decay. (Design: Built-in profiles)

**Robot vacuum worked example** (Design: Worked examples)

1. Scope: ask about floor types, rugs and thresholds, pets, budget, mopping or self-emptying, privacy (cameras, cloud accounts) and region, since model names differ by country. Answers become a priority weighting.
2. Plan: shortlist candidate models, then one sub-question per criterion: suction on carpet, edge cleaning, navigation, obstacle avoidance, noise, app and privacy, long-term reliability, running costs.
3. Gather: testers who publish their methods first, then long-term owner reports in forums, then hands-on video reviews. Manufacturer pages count only as specs.
4. Extract: claims carry product, criterion, test method and firmware, and are typed. Entity resolution merges regional names and revisions.
5. Weigh incentives: opinions from sources with detected incentives are discounted heavily. Their observations ("it got stuck on this rug at 4:12") still count.
6. Compare fairly: scores are compared only within one tester's method. Across testers, the system checks whether rankings agree. Prices and availability are fetched live at report time.
7. Report: weighted models × criteria matrix with evidence behind each cell, a "who it's for" summary per model, and a watch for new models, firmware changes and reliability reports.

## Implementation sketch

These are suggestions, not decisions.

- Incentive counselor signals: affiliate URL patterns and tracking params, disclosure phrases ("sponsored," "provided by," "affiliate"), MinHash near-duplicate overlap across listicles (reuse F23). Output an `incentive` flag per source with reasons.
- Discount math: `weight = kind_weight × (incentive && kind == opinion ? heavy_discount : 1)`. Put the discount value in the profile.
- Entity resolution: a `products` table with canonical id and aliases (regional names, revisions, SKU). Seed by Classifier-role normalization and confirm ambiguous merges in the UI.
- Priority weights: stored on the question as `{criterion: weight}` from scoping answers, and editable in plan review.
- Matrix cell = within-tester normalized scores aggregated across testers, plus an agreement flag when tester rankings disagree.
- Volatile facts: a `live_fact` type that the renderer fetches through the fetch proxy at report time and stamps with fetch time. It is never written into the ledger.
- Output template `comparison_matrix`: rows are models, columns are criteria, cells link to their evidence.

## Acceptance criteria

- [ ] Vacuum matrix discounts a sponsored "best of" verdict but keeps its footage as an observation
- [ ] Two regional names for the same model resolve to one product in the matrix.
- [ ] Changing a priority weight in plan review changes the matrix ranking without new fetches.
- [ ] Prices in the report come from a live fetch, show their fetch time, and don't appear in the claim ledger.
- [ ] Scores from two testers with different methods are never averaged directly. A disagreement in ranking is shown.
- [ ] Manufacturer pages produce only spec-kind claims.

## Open questions

- "Video transcripts" are a preferred corpus, but [F45](f45-video-transcripts.md) isn't a dependency. Should F44 wait for F45, or ship first and gain video later?
- Is there a sensible default for "heavy discount"? The design says heavy but gives no number.
- Region: should it come only from scoping, or also from an instance setting?
