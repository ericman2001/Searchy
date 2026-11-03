# F37 — Reliability judges and calibrated confidence labels

**Milestone:** M9 Source depth and adversarial review · **Status:** ⬜ Not started · **Depends on:** [F23](../m05-source-quality/f23-claim-confidence.md), [F26](../m06-async-research/f26-datasets-analyst.md), [F36](f36-provenance-tiers.md) · **Unblocks:** [F43](../m10-profiles-media/f43-science-health.md), [F44](../m10-profiles-media/f44-products-profile.md)

[← Back to plan](../PLAN.md)

## Goal

This feature finishes the source reliability score with the remaining judged signals: lateral reputation, evidence level, specificity, recency and conflict of interest. It turns support and contradiction into calibrated confidence labels (high, moderate, low, contested, unverified), each with stated reasons. It also adds ingestion checks for datasets. Users get three separate, explained badges on every claim instead of one opaque number.

## Scope

**In scope**
- Lateral reputation (SIFT): search for what other sources say about a domain or author, cached per domain and author.
- An evidence-level judge for studies: design, sample size, preregistration and data availability.
- A specificity judge on passages.
- Recency against question type.
- A conflict-of-interest judge (flag only).
- A reliability breakdown on the Sources screen that combines these with the F21 and F22 signals. Everything uses F21's per-project signal store and weighted score (PLAN.md decision D7).
- Calibrated claim labels with reasons, replacing or extending F23's corroboration counts.
- The "high" rule tunable in config.
- Dataset ingestion checks (schema, nulls, ranges), with failures shown on the dataset and on any chart built from it.

**Out of scope**
- Source-type classifier and domain trust list: [F21](../m05-source-quality/f21-source-classifier-trust.md).
- Retractions and citation impact: [F22](../m05-source-quality/f22-crossref-openalex.md).
- Independence detection: [F23](../m05-source-quality/f23-claim-confidence.md).
- Per-profile evidence hierarchies and GRADE-style certainty: [F43](../m10-profiles-media/f43-science-health.md).
- Incentive weighting by claim kind: [F44](../m10-profiles-media/f44-products-profile.md).
- Moving the "high" rule into the profile TOML: [F41](../m10-profiles-media/f41-profile-system.md).

## Design notes

**Reliability signals added here** (Design: Source quality and verification › 2. Source reliability)

| Signal | Where it comes from | Notes |
| --- | --- | --- |
| Evidence level | Judge extraction for studies | Design (meta-analysis, RCT, observational, case report), sample size, preregistration, data availability |
| Lateral reputation | Searches for what other sources say about this one (SIFT) | Cached per domain and author |
| Specificity | Judge on the passage | Named sources, verifiable details and linked data score up; "experts say" scores down |
| Recency | Publication date vs. question type | Matters for current-state questions, not history |
| Conflict of interest | Judge on funding and affiliation statements | Flag only; never auto-penalize |

Already present from earlier features: source type, retraction or correction (a retraction caps the score regardless of other signals), citation impact (normalized by field and age), domain trust, and content-farm signals.

**Lateral reading by depth** (Design: Research pipeline › Depth presets): off at quick, unknown domains only at standard, every source at thorough.

**Claim confidence** (Design: Source quality › 3. Claim confidence)
- Independence comes first (from F23).
- The earliest dated appearance of a claim is treated as its likely origin. Later copies add nothing.
- Support and contradiction, weighted by reliability and tier, map to calibrated labels in the style of IPCC and GRADE: **high, moderate, low, contested, unverified**.
- Starting rule, tunable: "high" needs at least two independent origins, one of them primary, and no unresolved contradiction from a source of similar reliability.

**Display** (Design: Source quality, intro): every claim carries three separate ratings: provenance distance, source reliability and corroboration. The UI shows all three as badges and doesn't blend them.

**Your own data** (Design: Source quality › Your own data)
- Notes are their own tier with a reliability you set per uploaded document or collection.
- Datasets record where they came from. Ingestion runs schema, null and range checks. Failures show on the dataset and on any chart built from it.

**Reliability storage and weights** (PLAN.md D7): signals are stored per source, per project. Each signal's weight is a domain-profile setting, with General defaults until F41. F21 owns the store and the combining formula; this feature only adds signals and weights.

**Model routing** (Design: Model routing): judges run in the classifier or verifier roles. Lateral searches go through the fetch proxy and SearXNG.

## Implementation sketch

Suggestions, not decisions.

1. Write the new signals into F21's `source_signals (project_id, source_id, …)` store (D7), adding `judged_by_model`. Add a `reputation_cache` keyed by domain or author, with a TTL. The cache can be shared across projects; the signal rows are per project.
2. Lateral reputation job: run a SearXNG query such as `"<domain>" reliability OR criticism OR fact-check`. A small model summarizes the stance and cites the passages it used. Respect the depth knob.
3. Evidence-level judge: structured output `{design, n, preregistered, data_available}`. Run it only for study source types.
4. Specificity and conflict-of-interest judges run per passage or per source and use the existing judge plumbing from F14.
5. Add default weights for the new signals to the General defaults:
   - `lateral_reputation = 1.0`
   - `evidence_level = 1.0` (studies only)
   - `specificity = 0.5`
   - `recency = 0.5` (applied only when the question type is current-state)
   - `conflict_of_interest = 0`, so it's a flag only, as the design requires.

   F21's formula does the combining and F22's caps still apply. Each component stays stored, so the Sources screen shows a breakdown with reasons.
6. Write a `confidence_label(claim)` function over the independent origins from F23, the tiers from F36 and reliability. Put the thresholds in a config struct that F41 can later load from profile TOML. Store the label and a reasons list.
7. Dataset checks: during F26 registration, infer the schema and run null and range checks with DuckDB. Store the results, and surface failures on the dataset page and on outputs that reference it.
8. UI: three badges on claims (tier, reliability, confidence), each with a tap-to-expand reasons list.

## Acceptance criteria

- [ ] Claims show high, moderate, low or contested badges with their reasons
- [ ] A claim with two independent origins, one primary, and no comparable contradiction is labeled high. Removing the primary one drops it below high.
- [ ] A claim with a contradiction from a source of similar reliability is labeled contested.
- [ ] A conflict-of-interest statement shows as a flag but leaves the reliability score unchanged.
- [ ] The same source scores differently in two projects when one project's profile, or its overrides, weights `lateral_reputation` differently.
- [ ] A second lookup for the same domain hits the lateral reputation cache and doesn't refetch.
- [ ] A CSV with nulls in a required column shows a failed check on the dataset and on a chart built from it.

## Open questions

- The design lists "unverified" as a label, but this feature's "Done when" lists only four. Treat unverified as the label for claims with no qualifying support.
- Resolved (D7): signals are stored per source, per project, and combined in F21 using domain-profile weights. Badges still show the breakdown.
- The "high" rule lives in the profile TOML per the design, but profiles arrive in [F41](../m10-profiles-media/f41-profile-system.md). Use built-in General defaults now, shaped like the future profile field (D7).
- Dataset ingestion checks depend on [F26](../m06-async-research/f26-datasets-analyst.md). The plan lists only F23 and F36 as dependencies, but F26 lands earlier in M6, so this is fine in practice.
