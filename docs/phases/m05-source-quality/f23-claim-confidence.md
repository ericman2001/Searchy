# F23 — Claim confidence and independence detection

**Milestone:** M5 Source quality · **Status:** ⬜ Not started · **Depends on:** [F13](../m03-ledger-verification/f13-claim-ledger.md), [F21](f21-source-classifier-trust.md) · **Unblocks:** [F28](../m07-idea-clouds-export/f28-claim-graph-clustering.md), [F35](../m08-project-chat/f35-output-staleness-derived.md), [F37](../m09-review/f37-reliability-judges-labels.md)

[← Back to plan](../PLAN.md)

## Goal

Make corroboration honest. Ten articles repeating one press release are one source, not ten. This feature groups sources into independent origins, counts support and contradiction per claim, and shows the result as the third badge on every claim.

## Scope

**In scope**

- Grouping supporting sources into origins: shared press release, wire syndication, near-duplicate text (MinHash), same author or organization, and citation loops.
- Earliest dated appearance treated as the likely origin.
- Per-claim corroboration counts: independent supporting origins and contradicting origins.
- A corroboration badge on claims in the ledger and in reports.
- Excluding derived passages from corroboration.

**Out of scope**

- Calibrated labels (high, moderate, low, contested, unverified): [F37](../m09-review/f37-reliability-judges-labels.md).
- Provenance tiers and citation chasing: [F36](../m09-review/f36-provenance-tiers.md).
- Contradicts edges on the idea map: [F28](../m07-idea-clouds-export/f28-claim-graph-clustering.md).
- Saved chat answers as derived notes: [F35](../m08-project-chat/f35-output-staleness-derived.md).

## Design notes

From Design: Source quality › 3. Claim confidence:

- Independence comes first. Sources collapse into one origin when they share:
  - a press release,
  - wire syndication,
  - near-duplicate text (MinHash),
  - the same author or organization,
  - or a citation loop (citogenesis: Wikipedia citing an article that took the fact from Wikipedia).
- The earliest dated appearance of a claim is treated as its likely origin. Later copies add nothing.
- Support and contradiction, weighted by reliability and tier, map to calibrated labels in the style of IPCC and GRADE. The labels arrive in F37; this feature produces the counts they're built on.
- Starting rule for "high", tunable in the profile TOML: at least two independent origins, one of them primary, and no unresolved contradiction from a source of similar reliability.

Other relevant notes:

- The three ratings are shown as separate badges, never blended into one number. (Design: Source quality)
- Derived passages are searchable and visible but never count toward corroboration. (Design: Source quality › 1. Provenance distance)
- Claim ledger: each claim has supporting and contradicting passages, a confidence, and the runs that touched it. (Design: Knowledge store › Organization)
- Reliability for weighting support and contradiction is the per-project score from [F21](f21-source-classifier-trust.md). Signals are stored per source, per project, with weights from the domain profile (PLAN.md decision D7). So each origin's weight is that project's score for its earliest member.
- News profile later adds "wire-syndication independence" and "outlet diversity". (Design: Domain profiles › Built-in profiles)

## Implementation sketch

Suggestions only.

1. Near-duplicates: compute MinHash signatures over passage shingles at ingestion. Use LSH buckets to find candidate pairs, then confirm with Jaccard similarity above a threshold.
2. Syndication: detect wire bylines and datelines (AP, Reuters, AFP and similar) and canonical-link tags pointing elsewhere.
3. Press releases: match against near-duplicate text from known press-release hosts, or a shared quote block.
4. Same author or organization: normalize author names and publisher domains from metadata.
5. Citation loops: when a passage cites a source that is later in time and itself cites back, collapse them.
6. Union-find over sources per claim gives `origin_id`. The member with the earliest date is the origin.
7. Tables: `claim_origins (claim_id, origin_id, stance, earliest_source_id, member_count)`. Recompute when a claim gains or loses a passage.
8. Weighting: store a `weight` on each `claim_origins` row, equal to the project's reliability score (D7) for the origin's earliest member source. A capped (retracted) source therefore contributes almost nothing. Recompute when that score changes.
9. Badge: "3 independent sources (7 total), 1 contradicting". Tap to see the grouping and why sources were merged.

## Acceptance criteria

- [ ] Two articles from one press release count once
- [ ] Two wire-syndicated copies of one story collapse to one origin.
- [ ] The earliest-dated source in a group is marked as the origin.
- [ ] A derived passage supporting a claim does not increase its independent count.
- [ ] The badge explains why sources were grouped.
- [ ] Counts update when a run adds a new supporting passage to an existing ledger claim.

## Open questions

- MinHash shingle size and similarity threshold. Tune these on real news data.
- The "high" rule needs provenance tier ("one of them primary"), which arrives in [F36](../m09-review/f36-provenance-tiers.md). Until then, only counts are shown.
- Resolved (D7): tunable values live in the domain profile. Before [F41](../m10-profiles-media/f41-profile-system.md), they're built-in General defaults using the keys the profile will own (for example `[confidence]` and `[independence]`), which F41 moves into `general.toml`.
