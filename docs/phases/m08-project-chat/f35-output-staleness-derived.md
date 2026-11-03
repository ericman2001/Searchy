# F35 — Output staleness and derived notes

**Milestone:** M8 Project chat and outputs · **Status:** ⬜ Not started · **Depends on:** [F23](../m05-source-quality/f23-claim-confidence.md), [F34](f34-outputs.md) · **Unblocks:** [F40](../m09-review/f40-verdict-card.md)

[← Back to plan](../PLAN.md)

## Goal

Keep outputs honest as the ledger changes, and stop the tool from corroborating itself. When a claim an output was built from changes, the output is marked stale and can be regenerated. Chat answers you save become derived notes: searchable, but never counted as independent support for any claim.

## Scope

**In scope**

- Marking outputs stale when any claim or query they were built from changes.
- Marking reports stale the same way.
- A stale marker and a regenerate button on the Outputs screen.
- Regeneration from the stored claim ids and query ids.
- "Save to project" on a chat answer, stored as a derived note.
- Excluding derived passages from corroboration and confidence calculations.

**Out of scope**

- Marking dependent reports and outputs stale when a review verdict is accepted: [F40](../m09-review/f40-verdict-card.md). F40 uses the mechanism built here.
- Derived-tier detection for AI content-farm pages: [F36](../m09-review/f36-provenance-tiers.md).
- Confidence calculation itself: [F23](../m05-source-quality/f23-claim-confidence.md).

## Design notes

- Every output is saved with the claim and query ids it was built from, so it can be regenerated, and it is marked stale when those claims change. (Design: Project chat › Outputs)
- Chat answers can be saved to the project, labeled as derived. Derived text is searchable but never counts as an independent source, so the tool can't corroborate itself. (Design: Project chat › Outputs)
- Provenance tier "Derived" means machine-generated text, such as saved chat answers and suspected AI content-farm pages. Derived passages are searchable and visible but never count toward corroboration. (Design: Source quality › Provenance distance)
- Outputs screen: "Saved charts, tables and summaries, with stale markers and a regenerate button." (Design: Web interface › Screens)
- Claims can be pinned, edited or rejected in the ledger, and the next run respects that. (Design: Knowledge store › Organization)
- Accepting a review verdict updates the ledger, adds contradicts edges and marks dependent reports and outputs stale. (Design: Adversarial review › The verdict card)

## Implementation sketch

Suggestions, not decisions:

1. Give each claim a `revision` counter (or `updated_at`) that bumps on edit, reject, pin, merge or confidence change. Store the revision each output saw in its `built_from` record.
2. Write a single `mark_dependents_stale(claim_ids)` function. Call it from every path that changes a claim: ledger edits, run merges and (later) verdict acceptance. Avoid scattered triggers.
3. Use a dependency table, e.g. `artifact_dependencies (artifact_kind, artifact_id, claim_id, revision)`, so reports and outputs share one mechanism.
4. Dataset outputs go stale when the dataset is re-ingested and its version changes.
5. Regenerate re-runs the output's recipe against current claims and creates a new version. Keep the old one for comparison.
6. Store saved chat answers as a source of kind `derived_note` with passages at the derived tier. Index them in F9 so they're searchable.
7. In F23's corroboration code, filter out derived-tier passages before counting origins. Add a regression test.

## Acceptance criteria

- [ ] Editing a claim marks its chart stale; a derived note never raises a claim's confidence
- [ ] Rejecting a claim marks every report and output that cites it stale. Outputs that don't cite it are unaffected.
- [ ] Regenerating a stale output produces a new version built from current claim revisions and clears the stale marker.
- [ ] A saved chat answer appears in search results labeled as derived.
- [ ] A derived note that repeats a claim word for word doesn't change that claim's corroboration count.
- [ ] Re-ingesting a dataset with changed values marks charts built from it stale.

## Open questions

- Should a confidence-label change alone (with no text edit) mark outputs stale? A chart may not depend on confidence, but a summary does. Probably yes for summaries and digests, and configurable for charts.
- Should regeneration be automatic for cheap outputs, or always manual? The design shows a button, so manual is the default.
- Should a derived note be citable in a report at all, or only visible in search and chat?
