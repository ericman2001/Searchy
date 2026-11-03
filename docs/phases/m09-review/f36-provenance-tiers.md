# F36 — Provenance tiers and citation chasing

**Milestone:** M9 Source depth and adversarial review · **Status:** ⬜ Not started · **Depends on:** [F13](../m03-ledger-verification/f13-claim-ledger.md), [F17](../m04-polite-web/f17-fetch-proxy.md), [F21](../m05-source-quality/f21-source-classifier-trust.md) · **Unblocks:** [F37](f37-reliability-judges-labels.md)

[← Back to plan](../PLAN.md)

## Goal

Every claim-passage pair gets a provenance tier that says how far the passage is from the original. When a passage attributes a claim to another work, a counselor queues that work so reports can cite the deepest tier found. Machine-generated text gets its own tier and can never corroborate anything. Users can then see whether an answer rests on original evidence or on a summary of a summary.

## Scope

**In scope**
- A provenance tier on each claim-passage link: primary, secondary, tertiary or derived.
- A tier classifier that judges each claim-passage pair.
- A citation-chasing counselor that detects attribution ("according to a study in…") and queues the cited work through the fetch proxy.
- The writer prefers the deepest tier found when it cites a claim.
- A derived tier for suspected AI content-farm pages, using content-farm signals.
- Derived passages are excluded from corroboration counts.
- A tier badge on citations and claims in the UI.

**Out of scope**
- Reliability scoring beyond content-farm signals, and calibrated labels: [F37](f37-reliability-judges-labels.md).
- Saved chat answers as derived notes: [F35](../m08-project-chat/f35-output-staleness-derived.md). This feature should reuse the same derived tier.
- Per-depth tuning of chasing (off, one hop, until primary) beyond a simple config knob: wired through presets in [F25](../m06-async-research/f25-fan-out-budget.md). This feature only exposes the knob.
- Profile-specific meanings of "primary": [F41](../m10-profiles-media/f41-profile-system.md), [F43](../m10-profiles-media/f43-science-health.md), [F44](../m10-profiles-media/f44-products-profile.md).

## Design notes

**Tiers** (Design: Source quality and verification › 1. Provenance distance)

| Tier | Meaning | Examples |
| --- | --- | --- |
| Primary | Original data, text or firsthand account | A study's own results, a statute, a dataset, a quote in the outlet that did the interview |
| Secondary | Reports on or analyzes primaries | Review articles, news coverage of a study, analyst reports |
| Tertiary | Summarizes secondaries | Wikipedia, textbooks, explainers |
| Derived | Machine-generated text | Saved chat answers, suspected AI content-farm pages |

- The tier belongs to the claim-passage pair, not the source. A news article is primary for its own interview quote and secondary for a study it describes.
- Citation chasing: when a passage attributes a claim to another work, a counselor queues that work. Reports cite the deepest tier found.
- Derived passages are searchable and visible but never count toward corroboration.

**Content-farm and AI signals** (Design: Source quality › 2. Source reliability): no author, templated structure, very high publish rate, no outbound citations. These are a weak down-weight plus the derived tier. Detectors are unreliable, so these signals never cause an automatic exclusion.

**Depth knob** (Design: Research pipeline › Depth presets): citation chasing is off at quick, one hop at standard, and runs until a primary source at thorough.

**Counselors** (Design: Research pipeline › How the chat harness stages map): citation chasing is a mid-turn counselor, alongside the source-quality gate and the loop guard. Chased fetches go through the polite fetch proxy like any other fetch.

**Your own data** (Design: Source quality › Your own data): uploaded notes are their own tier, with a reliability set per document or collection (D6). Treat "your notes" as a separate label next to the provenance tier, not as one of the four tiers.

## Implementation sketch

Suggestions, not decisions.

1. Add a `provenance_tier` column (enum) to the claim-passage link table from F13, plus `tier_reason` text and `chased_from_passage_id` (nullable).
2. Write a tier classifier in the classifier role. Run rules first: dataset or statute source types are primary, and reference sources such as Wikipedia default to tertiary. Fall back to a small model that sees the claim, the passage and the source type from F21.
3. Write an attribution detector over extracted passages. It finds DOIs, links, "according to", and named study or report references, then resolves them to a URL or DOI. Crossref and OpenAlex lookups from F22 are helpful here, though F22 isn't a hard dependency.
4. Build a `CitationChaseCounselor` that enqueues fetch jobs for resolved targets. It keeps a hop counter per chain and stops at the depth limit, at a primary tier, or when it hits a loop it has already visited. Respect the budget counselor.
5. When the chased work is read and extracted, link the matching claim to the new passage. The writer's citation selection orders candidates primary > secondary > tertiary > derived.
6. Compute content-farm signals per source and store them on the source row. A suspected content farm sets derived on its pairs and applies a weak reliability down-weight.
7. Update the corroboration counting from F23 to skip derived pairs.
8. Add tier badges in report citations (F16) and on the Sources and claim views.

## Acceptance criteria

- [ ] A news passage citing a study queues the study, and the report cites it at primary tier
- [ ] One news article is tagged primary for its own interview quote and secondary for a study it describes.
- [ ] At standard depth, chasing stops after one hop; at thorough, it continues until it reaches a primary source or runs out of targets.
- [ ] A citation loop, such as A citing B citing A, terminates without refetching.
- [ ] A page with no author, templated structure and no outbound citations is tagged derived, stays searchable, and does not raise any claim's corroboration count.
- [ ] A paywalled chased target is recorded as metadata only and the chain ends there.

## Open questions

- The citation-chasing counselor plugs into the counselor hooks from [H8](../mh-pipeline-core/h08-counselors.md).
- "Primary" means different things per domain: study data for science, direct testing or ownership for products. Until [F41](../m10-profiles-media/f41-profile-system.md) lands, use the general definition and keep the rule pluggable.
- The design doesn't say how a tier is shown when one claim has several passages at different tiers. Suggestion: show the deepest tier, and list the rest on the claim detail view.
