# F13 — Claim extraction and claim ledger

**Milestone:** M3 Ledger and verification · **Status:** ⬜ Not started · **Depends on:** [F12](../m02-local-knowledge/f12-quick-depth-e2e.md) · **Unblocks:** [F14](f14-verification-stage.md), [F23](../m05-source-quality/f23-claim-confidence.md), [F36](../m09-review/f36-provenance-tiers.md)

[← Back to plan](../PLAN.md)

## Goal
Turn passages into structured claims, each tied to the exact passage span that supports it. Keep them in a per-project claim ledger that grows across runs. This makes projects accumulate knowledge: the tenth question reuses what the first nine found, and you can pin, edit or reject claims so later runs respect your judgment.

## Scope
**In scope**
- An Extractor role step in the Read and extract stage: passage → candidate claims, each with a passage span.
- A claim ledger per project: claims with supporting and contradicting passages, a confidence placeholder, and the runs that touched them.
- Merging new candidate claims with matching claims already in the ledger.
- The writer now drafts from claims, still citing passage ids.
- A ledger lookup at the Scope stage: existing claims relevant to the question are loaded first.
- Pin, edit and reject actions on claims, recorded with the user who made them. Later runs respect these.
- A basic ledger view on the Project page.

**Out of scope**
- Confidence labels and independence detection → [F23](../m05-source-quality/f23-claim-confidence.md), [F37](../m09-review/f37-reliability-judges-labels.md).
- Provenance tiers per claim-passage pair → [F36](../m09-review/f36-provenance-tiers.md).
- Profile-specific claim schemas (PICO, product/criterion) → [F41](../m10-profiles-media/f41-profile-system.md), [F43](../m10-profiles-media/f43-science-health.md), [F44](../m10-profiles-media/f44-products-profile.md).
- Graph edges and clustering → [F28](../m07-idea-clouds-export/f28-claim-graph-clustering.md).
- Verification → [F14](f14-verification-stage.md).

## Design notes
**Ledger** (Design: Knowledge store › Organization)
- Project → questions → runs.
- The claim ledger is the project's running "what we know" list. Each claim has supporting and contradicting passages, a confidence, and the runs that touched it.
- You can pin, edit or reject claims, and the next run respects that.

**Pipeline** (Design: Research pipeline)
- Scope: check the question against the project's existing claim ledger.
- Read and extract: reader workers turn documents into passages and extract candidate claims, each tied to a passage span.
- Score: claims are merged with matching claims already in the ledger.
- Synthesize: the writer drafts the report from the claims.

**Extractor role** (Design: Model routing): turns passages into structured claims. Small or mid model; the highest-volume role. It runs on small fast models at every depth.

**Claim kinds** (Design: Domain profiles): measurement, observation, spec, opinion, mechanism. These are shared across profiles. The `kind` field can be stored now, even though weighting comes later.

**Merging must respect structure** (Design: Worked examples › Coffee): a finding in older Japanese adults must never merge into "coffee lowers mortality." Merging must not collapse claims with different scopes.

**Attribution** (Design: Teams and permissions): every ledger edit records who made it. Editors can edit the ledger; viewers can't.

## Implementation sketch
Suggestions only.
- Tables:
  - `claims(id, project_id, text, kind, fields_json, status[active|pinned|rejected], confidence NULL, created_run_id, updated_at)`
  - `claim_evidence(claim_id, passage_id, span_start, span_end, stance[supports|contradicts], run_id)`
  - `claim_runs(claim_id, run_id)`
  - `claim_edits(claim_id, user_id, action, before_json, after_json, at)`
- Extractor output schema (JSON): `{text, kind, span: {start, end}, scope: {population?, timeframe?, entity?}}`. Validate that the span is within the passage, and that the quoted span text actually appears in it.
- Merge: embed the claim text. Take the top-k ledger candidates above a similarity threshold, then have a small model judge "same claim / narrower / different". Merge only on "same claim". Keep the threshold in config.
- Rejected claims are excluded from synthesis and from future merges, so a rejected claim isn't recreated. Pinned claims are always loaded into synthesis context when relevant. Edited text replaces the claim's text, with the evidence kept.
- Run `extract` in the reader worker, which has no internet route. It calls models through the hub's model gateway on the internal network (D4; [F8](../m02-local-knowledge/f08-model-routing.md)).

## Acceptance criteria
- [ ] Two runs on related questions share ledger claims
- [ ] Every claim has at least one evidence row whose span text appears verbatim in the passage.
- [ ] Two claims that differ only in population (for example, "in older Japanese adults" versus "in adults") are not merged.
- [ ] A rejected claim isn't recreated or cited by the next run on a related question.
- [ ] A pinned claim is used in synthesis on the next relevant run.
- [ ] Ledger edits show who made them, and viewers can't edit.

## Open questions
- What "matching" means before profile claim schemas exist ([F41](../m10-profiles-media/f41-profile-system.md)): the generic `scope` fields are a stopgap. Expect a schema migration when profiles land.
- Resolved (D4): extraction runs in the reader worker and reaches models through the hub's model gateway over the internal network.
- The cost of running extraction on every retrieved passage at the highest-volume role. Consider extracting only from the top-ranked passages.
