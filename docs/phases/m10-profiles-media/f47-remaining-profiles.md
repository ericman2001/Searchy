# F47 — Remaining built-in profiles

**Milestone:** M10 Domain profiles and media · **Status:** ⬜ Not started · **Depends on:** [F26](../m06-async-research/f26-datasets-analyst.md), [F41](f41-profile-system.md) · **Unblocks:** —

[← Back to plan](../PLAN.md)

## Goal

This completes the built-in profile set: news and events, legal and policy, history and humanities, technical docs, and finance and companies. With these, most questions get domain-appropriate evidence rules out of the box. Each profile ships with a sample question that proves its rules work.

## Scope

**In scope**

- `news.toml`, `legal.toml`, `history.toml`, `technical_docs.toml`, `finance.toml`, each with every F41 profile field filled.
- Profile-specific rules listed below, implemented as counselors, extraction schema fields or reviewer checklist items.
- Notices: not-legal-advice and not-financial-advice.
- One sample question per profile, with a test that runs it and passes its own reviewer checklist.

**Out of scope**

- Science and health: [F43](f43-science-health.md). Products: [F44](f44-products-profile.md).
- The Climate and Software overlays of science. They aren't named in the F47 row; see Open questions.

## Design notes

**Built-in profiles** (Design: Domain profiles › Built-in profiles)

| Profile | For | Primary sources | Distinctive rules |
| --- | --- | --- | --- |
| News and events | What happened and what's happening | Official documents, on-the-scene reporting | Very fast decay, wire-syndication independence, corrections tracking, outlet diversity |
| Legal and policy | Laws, regulations, rulings | Statutes, regulations, court opinions | Jurisdiction and effective date on every claim, superseded-law checks, not-legal-advice notice |
| History and humanities | Past events, texts, ideas | Archival documents, contemporaneous accounts | No decay, dating and authorship checks, historiography |
| Technical docs | How to do X with tool Y | Official docs, source code, changelogs, issue trackers | Version on every claim; claims can be tested by running code in an Analyst worker |
| Finance and companies | Companies, markets, filings | Regulatory filings, earnings reports | Volatile figures fetched live, heavy incentive weighting, not-financial-advice notice |

**Related rules elsewhere in the design**

- Independence: sources sharing a press release or wire syndication collapse into one origin. (Design: Source quality › Claim confidence)
- Provenance: a quote in the outlet that did the interview is primary; a statute is primary. (Design: Provenance distance)
- Reviewer status check: laws, standards or guidelines that were later superseded. (Design: Adversarial review)
- Recency matters for current-state questions, not history. (Design: Source reliability)
- Volatile facts like prices are fetched live at report time, never stored as claims. (Design: Worked examples › Vacuums)
- Local corpora such as Stack Exchange dumps help with technical questions; Project Gutenberg helps with historical and literary text. (Design: Being a good netizen)
- The Analyst worker runs offline with tmpfs scratch. (Design: Architecture, Deployment)

## Implementation sketch

These are suggestions, not decisions.

| Profile | Claim schema additions | Decay | Reviewer checklist highlights | Output template |
| --- | --- | --- | --- | --- |
| News | event, date, place, actor | days | syndication check, corrections, outlet diversity | timeline + sourced summary |
| Legal | jurisdiction, effective date, instrument, section | none, but supersession checked | superseded law, jurisdiction mismatch | provision-by-provision table |
| History | date (with certainty), author, source date | none | dating, authorship, historiography (how interpretations changed) | narrative with source criticism notes |
| Technical docs | tool, version | months | version mismatch, deprecated APIs | how-to with tested snippets |
| Finance | company, period, metric, filing id | days for figures | incentive, stale figures | filing-backed fact table |

- Corrections tracking (news): when an outlet publishes a correction, link it to the original passage and down-rate the corrected claim.
- Outlet diversity (news): warn when the support comes from one outlet or one political lean.
- Code-tested claims (technical docs): a claim like "flag X does Y in v2.3" becomes a small script that the Analyst worker runs in a no-network container. The result is stored as a measurement-kind passage. This needs [F26](../m06-async-research/f26-datasets-analyst.md)'s Analyst worker.
- Live figures (finance): reuse F44's `live_fact` mechanism for prices and market data.
- Sample questions, for example: news "What happened at <recent event>?"; legal "What does GDPR Article 17 require?"; history "Who wrote the Federalist Papers, and how do we know?"; technical "How do I enable WAL mode in SQLite?"; finance "What was <company>'s revenue last fiscal year?"

## Acceptance criteria

- [ ] Each ships with a sample question that passes its own reviewer checklist
- [ ] All five profiles load and validate at startup, and every F41 field is set or inherited.
- [ ] Legal claims without jurisdiction or effective date are flagged as incomplete.
- [ ] Two news articles from the same wire story count as one origin.
- [ ] A technical-docs claim is tested by running code in an Analyst worker, and the result appears as a cited measurement.
- [ ] Legal and finance reports show their notices.

## Open questions

- The Climate and Software overlays appear in the built-in table but aren't assigned to any feature. They could be added here or as a follow-up.
- Technical docs needs the Analyst worker ([F26](../m06-async-research/f26-datasets-analyst.md)), but F26 isn't a listed dependency. It will be done by this point in practice.
- This feature is large for one session. Consider splitting it into one session per profile.
