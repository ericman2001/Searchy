# F21 — Source-type classifier and domain trust list

**Milestone:** M5 Source quality · **Status:** ⬜ Not started · **Depends on:** [F17](../m04-polite-web/f17-fetch-proxy.md), [H8](../mh-pipeline-core/h08-counselors.md) · **Unblocks:** [F22](f22-crossref-openalex.md), [F23](f23-claim-confidence.md), [F36](../m09-review/f36-provenance-tiers.md)

[← Back to plan](../PLAN.md)

## Goal

Label every source with what kind of thing it is and how much to trust its domain. This is the first visible piece of source scoring: the Sources screen shows type and trust for each source, with the reasons, and users can override trust per project.

## Scope

**In scope**

- A source-type classifier using URL, metadata and content.
- An editable domain trust list, seeded from Wikipedia's perennial sources list and DOAJ membership.
- Per-project trust overrides.
- The per-project reliability store and score (PLAN.md decision D7):
  - Signals are stored per source, per project.
  - They combine into one reliability score using weights from the domain profile.
  - Until [F41](../m10-profiles-media/f41-profile-system.md) exists, General defaults supply the weights.
  - F22 and F37 add their signals to this store.
- Notes as their own source type, with a reliability the user sets on uploaded documents. Folders were dropped (D6).
- The Sources screen: list, type, trust, reasons, archive link and override controls.

**Out of scope**

- Retraction and citation-impact signals: [F22](f22-crossref-openalex.md).
- Corroboration and independence: [F23](f23-claim-confidence.md).
- Provenance tiers, content-farm signals: [F36](../m09-review/f36-provenance-tiers.md).
- Lateral reputation, evidence level, specificity and calibrated labels: [F37](../m09-review/f37-reliability-judges-labels.md).
- Profile-specific trust rules: [F41](../m10-profiles-media/f41-profile-system.md).

## Design notes

- Every claim carries three ratings, shown as separate badges and never blended: provenance distance, source reliability and corroboration. This feature starts **source reliability**. (Design: Source quality and verification)
- Reliability signals owned here (Design: Source quality › 2. Source reliability):

| Signal | Where it comes from | Notes |
| --- | --- | --- |
| Source type | Classifier on URL, metadata and content | Peer-reviewed, preprint, government, standards body, news, reference, blog, forum, your notes |
| Domain trust | Your editable list, seeded from Wikipedia's perennial sources list and DOAJ membership | Per-project overrides |

- The classifier role handles source type, with rules first where possible, then a small model. (Design: Model routing › Roles)
- Sources are shared across projects with one archive copy, but trust overrides and notes are per project. (Design: Knowledge store › Organization)
- Notes are their own tier, with a reliability you set per uploaded document or collection, labeled "your notes" in citations. (Design: Source quality › Your own data; Knowledge store)
- Sources screen: every source with its tier, reliability breakdown, last-reviewed date, archive copy and override controls. (Design: Web interface › Screens)
- Settings holds the domain trust list. (Design: Web interface › Screens)
- Counselors include a source-quality gate. (Design: Research pipeline › harness stage mapping)
- Reliability storage and weights (PLAN.md D7):
  - Reliability signals are stored per source, per project. The same archived source can score differently in two projects, through overrides and profiles.
  - The weight of each signal is a domain-profile setting.
  - The score is a weighted combination of the signals present. Caps are applied after weighting: a cap is a ceiling that no other signal can lift. "A retraction caps the score" ([F22](f22-crossref-openalex.md)) is one such cap.
  - Badges still show the breakdown, not just the number.
- Admins manage instance-wide settings. Project owners and editors act within a project. (Design: Teams and permissions › Roles)

## Implementation sketch

Suggestions only.

1. Tables:
   - `sources.source_type`, `sources.type_reason`
   - `domain_trust (domain, score, label, origin)`, where origin is `seed:perennial`, `seed:doaj` or `user`
   - `project_trust_overrides (project_id, domain_or_source_id, score, reason, by_user, at)`
2. Seed data: ship a snapshot of the perennial sources list (reliable / mixed / unreliable / deprecated) and DOAJ journal domains as a data file. Record the snapshot date. Import at migration time.
3. Classifier:
   - Rules first: TLDs (`.gov`), known publishers, DOI presence, arXiv/bioRxiv for preprints, forum and blog platforms.
   - Then the classifier-role model on metadata and the first part of the content when rules don't decide.
   - Store the reason string either way.
4. Reliability store (D7):
   - Table: `source_signals (project_id, source_id, signal, value REAL 0–1, cap REAL NULL, reason, raw_json, computed_at)`, one row per signal.
   - Cache the result in `source_reliability (project_id, source_id, score, capped_by, profile_id, weights_version, computed_at)`.
   - Score: `score = Σ(wᵢ·vᵢ) / Σ(wᵢ)` over the signals present, then `score = min(score, min(caps))`. Record which signal capped it in `capped_by`.
   - Weights come from the domain profile's `[reliability.weights]` table. Before F41, they come from a built-in General default, a Rust constant shaped like that TOML table, which F41 moves into `general.toml`.
   - Recompute on signal change, override change or profile change.
   - Starting weights: `source_type = 1.0`, `domain_trust = 1.0`. F22 and F37 add theirs.
5. Sources screen: htmx list with filters by type. Each row shows type, trust, the reasons and a link to the archive. Owners and editors can override trust per project.
6. Settings: an admin page to edit the instance trust list.

## Acceptance criteria

- [ ] Sources show type and trust in the Sources screen
- [ ] A `.gov` URL, a DOI-bearing journal page, an arXiv page and a forum thread are each classified correctly by the rules pass.
- [ ] Every classification stores a human-readable reason.
- [ ] A per-project trust override changes trust in that project only.
- [ ] A domain marked unreliable in the seed list shows that label with its origin.
- [ ] Notes show as "your notes" with the reliability set on the document.
- [ ] Signals are stored per (project, source). Changing a weight in the active General defaults recomputes the score, and the Sources screen shows the breakdown and the combined score.

## Open questions

- Resolved (D7): signals are stored per source, per project, and combined with weights that come from the domain profile (General defaults until F41). The weighted mean with caps is a starting formula to tune.
- License and refresh schedule for the perennial sources and DOAJ seed data.
- The source-quality gate counselor plugs into the counselor hooks from [H8](../mh-pipeline-core/h08-counselors.md), and the source-type classifier into the classifier registry from [H6](../mh-pipeline-core/h06-stage-runner-classifiers.md) and [H10](../mh-pipeline-core/h10-judges.md).
