# F22 — Crossref retractions and OpenAlex metadata

**Milestone:** M5 Source quality · **Status:** ⬜ Not started · **Depends on:** [F21](f21-source-classifier-trust.md) · **Unblocks:** [F38](../m09-review/f38-review-job.md)

[← Back to plan](../PLAN.md)

## Goal

Add scholarly signals to source reliability. Crossref tells us whether a paper was retracted or corrected, and a retraction caps the score no matter what else is true. OpenAlex provides metadata and citation impact, normalized so new papers aren't punished.

## Scope

**In scope**

- A Crossref client: look up DOIs, read retraction, correction and expression-of-concern updates (including Retraction Watch data).
- An OpenAlex client using the polite pool (`mailto`), for metadata and citation counts.
- Retraction cap and visible flag on sources and their claims.
- Citation impact normalized by field and age.
- DOI detection from URLs, page metadata and PDFs.

**Out of scope**

- Status-check watches that re-run retraction checks: [F40](../m09-review/f40-verdict-card.md).
- The reviewer's status-check step: [F38](../m09-review/f38-review-job.md).
- An OpenAlex local snapshot corpus. This is a gap item (other local corpora) in [PLAN.md](../PLAN.md#design-items-with-no-feature-yet).
- Evidence-level judging of studies: [F37](../m09-review/f37-reliability-judges-labels.md).

## Design notes

- Reliability signals owned here (Design: Source quality › 2. Source reliability):

| Signal | Where it comes from | Notes |
| --- | --- | --- |
| Retraction or correction | Crossref metadata (includes Retraction Watch data) | A retraction caps the score regardless of other signals |
| Citation impact | OpenAlex | Normalized by field and age so new papers aren't punished |

- OpenAlex: a snapshot, or the API with a `mailto` for the polite pool. Used for scholarly metadata and citation counts. (Design: Being a good netizen › Local corpora)
- Prefer APIs and bulk dumps over scraping. All network calls go through the Fetcher. (Design: Being a good netizen › Fetch proxy)
- The reviewer's status check later reuses this: retractions, corrections, expressions of concern and errata from Crossref. (Design: Adversarial review › checklist step 1)
- BibTeX or Zotero imports bring sources with their metadata. DOIs from those can feed this lookup. (Design: Knowledge store)

## Implementation sketch

Suggestions only.

1. Crossref: `GET https://api.crossref.org/works/{doi}` through the Fetcher, with the `mailto` in the User-Agent. Read `update-to` / `updated-by` relations for retraction, correction and expression-of-concern notices.
2. OpenAlex: `GET https://api.openalex.org/works/doi:{doi}?mailto=...` for title, authors, venue, year, concepts or topics, and `cited_by_count`.
3. Normalization: compare `cited_by_count` with a field-and-year baseline (OpenAlex percentile fields if available, otherwise a stored baseline table). Store a percentile, not raw counts.
4. Tables:
   - `scholarly_meta (source_id, doi, openalex_id, venue, year, field, citation_pct, fetched_at)`
   - `source_status (source_id, kind, notice_doi, date, origin)`, where kind is retracted, corrected, concern or erratum
5. Write both signals into F21's per-project signal store, `source_signals` ([F21](f21-source-classifier-trust.md), PLAN.md decision D7):
   - `citation_impact`, with value = `citation_pct`. Its default weight is 0.5.
   - `status`, with value = 1.0 when clean, plus a `cap` taken from the profile's `[reliability.caps]` table.
   - Starting General caps are `retracted = 0.05`, `concern = 0.4`, and no cap for `corrected` and `erratum` (they get a note only).
   - Because F21 applies caps after weighting, a retracted paper's score is at most 0.05 however high its other signals are. `capped_by = "status:retracted"` drives the red flag.
   - Status is a property of the paper, so the same status row feeds every project's signal store. Weights and caps can still differ by profile.
6. UI: a red "Retracted" flag on the source row and on every citation to it, with a link to the notice.
7. Cache responses. Use the Fetcher's HTTP cache and a TTL for re-checks.

## Acceptance criteria

- [ ] Known retracted paper is capped and flagged
- [ ] A DOI is detected from a publisher URL, from page metadata and from an imported BibTeX entry.
- [ ] OpenAlex requests include the `mailto` parameter.
- [ ] A corrected (not retracted) paper shows a correction note without the cap.
- [ ] A retracted paper with top-percentile citations and a trusted domain still scores ≤ the `retracted` cap (0.05 by default), with `capped_by = status:retracted`.
- [ ] Two papers with equal raw citations but different ages get different normalized impact.
- [ ] All Crossref and OpenAlex calls go through the Fetcher and are cached.

## Open questions

- Resolved (D7): caps and weights live in the domain profile, with General defaults until F41. The retraction cap starts at 0.05, to tune.
- Whether to build the OpenAlex snapshot now or stay on the API. The design allows both.
- Preprints whose published version changed the results are a reviewer check ([F38](../m09-review/f38-review-job.md)). Linking preprint to published version may fit here.
