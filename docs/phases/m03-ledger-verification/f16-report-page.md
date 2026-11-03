# F16 — Report page with tappable citations

**Milestone:** M3 Ledger and verification · **Status:** ⬜ Not started · **Depends on:** [F14](f14-verification-stage.md) · **Unblocks:** [F30](../m07-idea-clouds-export/f30-export-backup.md)

[← Back to plan](../PLAN.md)

## Goal
The Report screen is where you read the answer. Every citation is tappable and shows the exact passage, its source and its score, right there on your phone. Verification flags are visible inline, so you can see what is solid and what shipped unverified. This is the page that makes Searchy's core promise visible.

## Scope
**In scope**
- A Report page, rendered on the server with askama and htmx, in a phone-first single column with large tap targets.
- Inline citations. Tapping one opens a sheet or panel with the passage text (cited span highlighted), the source title and type, a "your notes" label where it applies, and the source's score badges (placeholders until M5/M9).
- Visible verification status per sentence: verified, partial or unverified, and uncited.
- Links from a passage to the source detail and to the claim in the ledger.
- A text-selection hook ("select text to Ask or Review") that shows the actions but leaves them stubbed.
- Working well on a slow connection: the passage panel loads lazily with an htmx request.

**Out of scope**
- The archived-copy link → [F18](../m04-polite-web/f18-warc-archive.md). Leave a slot for it.
- Real reliability, tier and confidence badges → [F21](../m05-source-quality/f21-source-classifier-trust.md), [F23](../m05-source-quality/f23-claim-confidence.md), [F36](../m09-review/f36-provenance-tiers.md), [F37](../m09-review/f37-reliability-judges-labels.md).
- The Ask action → [F31](../m08-project-chat/f31-project-chat-loop.md)/[F33](../m08-project-chat/f33-scoped-chat.md). The Review action → [F38](../m09-review/f38-review-job.md).
- Comments on reports (listed as a gap in PLAN.md).
- Export → [F30](../m07-idea-clouds-export/f30-export-backup.md).

## Design notes
**Screen** (Design: Web interface › Screens): Report shows the answer with inline citations. Tap a citation to see the exact passage and source ratings. Select text to Ask or Review.

**UI rules** (Design: Web interface)
- Server-rendered HTML from the Rust hub, with htmx for interactivity.
- Phone-first: one column, large tap targets, and pages that work on a slow connection.
- No build step and no JavaScript framework.

**Badges** (Design: Source quality and verification): every claim carries three separate ratings: provenance distance, source reliability and claim confidence. The UI shows all three as badges instead of blending them into one number. Lay out the badge slots now, even before the values exist.

**Citations to your material** (Design: Knowledge store): labeled "your notes".

**Unverified** (Design: Verification): after the retry limit, unsupported claims ship marked as unverified.

**Permissions** (Design: Teams and permissions): viewers can read reports. Use the project role checks from F5.

## Implementation sketch
Suggestions only.
- Route `GET /projects/:p/runs/:r/report`. Render the stored report as structured sentences, not raw Markdown, so each sentence carries its verification status and citations.
- Citation markup: `<button hx-get="/passages/:id/card?run=:r" hx-target="#sheet">[n]</button>`. The card template shows the passage with the span highlighted, the source block, badge slots and links.
- Sheet: a bottom sheet on narrow screens and a side panel on wide ones, using CSS only with no JavaScript framework. Make sure it can be closed with a thumb and with the Back button (`hx-push-url`).
- Sentence status styling: a subtle underline or icon for partial and unverified, with a legend at the top of the report. Don't rely on color alone.
- Text selection: a small vanilla-JS snippet shows a floating "Ask / Review" bar on selection and posts the selected range. Keep the handlers stubbed until F31/F38.
- Test on a real phone over a throttled connection (Chrome DevTools "Slow 4G" as a proxy).

## Acceptance criteria
- [ ] Works one-handed on a phone
- [ ] Tapping a citation shows the cited passage with its span highlighted, the source and the badge area, without a full page reload.
- [ ] Unverified and partial sentences are visibly marked, and the marks are explained in a legend.
- [ ] Citations to notes show the "your notes" label.
- [ ] A user without access to the project gets a 403 or 404 for the report and its passage cards.
- [ ] The page stays usable with a throttled "Slow 4G" profile. The first render doesn't wait on passage cards.

## Open questions
- Exact placement of the three badges on a 380 px screen. A quick mockup before coding would help.
- The text-selection UX on mobile, where native selection menus compete with a custom bar.
