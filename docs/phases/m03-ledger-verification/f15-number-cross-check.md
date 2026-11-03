# F15 — Number cross-check

**Milestone:** M3 Ledger and verification · **Status:** ⬜ Not started · **Depends on:** [F14](f14-verification-stage.md), [H11](../mh-pipeline-core/h11-calculator-cross-check.md) · **Unblocks:** —

[← Back to plan](../PLAN.md)

## Goal
Catch misquoted numbers mechanically. Models are bad at copying figures exactly, and a judge model can easily wave through "23%" when the passage says "32%". For every number in a cited sentence, the harness re-reads the value from the cited passage itself and compares, without relying on a model's opinion.

## Scope
**In scope**
- Extracting numbers from report sentences: plain numbers, percentages, ranges, ratios and common units.
- Finding the matching value in each cited passage and comparing them, with tolerance for rounding and formatting.
- A mismatch counts as a verification failure: it triggers a retry through F14 with the reason "figure X not found in passage; passage says Y".
- Recording the result per number in the verification records.

**Out of scope**
- Values in charts → [F34](../m08-project-chat/f34-outputs.md). It reuses this code with the rule "numbers come from passages or queries, never from the model's prose".
- Dataset query numbers → [F26](../m06-async-research/f26-datasets-analyst.md).
- Unit conversion beyond simple formatting. Note it as an open question if needed.

## Design notes
**Numbers** (Design: Source quality and verification › Verification): numbers get the calculator-example treatment from the chat harness. The harness re-reads the number from the passage itself and compares. The calculator example and its `computed_answer` cross-check are built in [H11](../mh-pipeline-core/h11-calculator-cross-check.md); this feature applies the same pattern to passages.

**Same principle later** (Design: Project chat › Outputs): numbers in charts come from passages or queries, never from the model's prose; the harness reads the values itself. Write the matcher so F34 can reuse it.

**Failure path** (Design: Research pipeline step 7): failures go back to Synthesize with the reasons, up to the retry limit, then ship with visible flags.

## Implementation sketch
Suggestions only.
- Module `verify::numbers`.
- `extract_numbers(text) -> Vec<NumMention { value, unit?, kind: plain|percent|range|ratio, span }>`. Handle thousands separators, decimals, "1.2 million", "3–5", negatives, and `%` versus "percent". Handle "twice"/"half" only if cheap.
- `match_in_passage(mention, passage) -> Match { found: exact|rounded|none, passage_value?, passage_span? }`. Rounded means equal after rounding the passage value to the sentence's precision. For example, a passage value of 32.4% matches "32%".
- Deterministic first. If a sentence number has no literal counterpart but the passage has nearby numbers, record `none` with the candidate values listed in the reason, so the writer can fix it.
- Derived numbers, such as a difference computed from two passage values, are out of scope for the first cut. Flag them as "derived, not checked" instead of failing them.
- Hook this into F14 as a pre-pass on each (sentence, passage) pair. A `none` result counts as unsupported for retry purposes. Reuse the comparison and trace format from H11's `computed_answer` validator where it fits.

## Acceptance criteria
- [ ] Altered figure in a draft triggers a retry
- [ ] "32%" in a sentence matches a passage saying "32.4 percent".
- [ ] "23%" in a sentence citing a passage that says "32%" fails, with a reason naming both values.
- [ ] Ranges ("3–5 cups") and magnitudes ("1.2 million") are parsed and matched.
- [ ] A sentence number with no counterpart in any cited passage is flagged in the stored verification record after the retry limit.

## Open questions
- How strict to be on rounding. "About a third" versus 34% is arguably fine, but harder to check deterministically.
- Derived or computed numbers (differences, ratios across passages) are common in comparisons. Decide whether to check them later or always label them.
- Locale formatting (`1.234,5`) in non-English sources.
