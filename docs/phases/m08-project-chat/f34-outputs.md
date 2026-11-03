# F34 — Outputs: charts, tables, summaries, timelines

**Milestone:** M8 Project chat and outputs · **Status:** ⬜ Not started · **Depends on:** [F26](../m06-async-research/f26-datasets-analyst.md), [F28](../m07-idea-clouds-export/f28-claim-graph-clustering.md), [F31](f31-project-chat-loop.md) · **Unblocks:** [F35](f35-output-staleness-derived.md)

[← Back to plan](../PLAN.md)

## Goal

Let chat produce charts, tables, summaries, timelines and contradiction digests from what the project already knows. Every number comes from a passage or a dataset query, never from the model's prose. Every output is saved with the ids it was built from, so it can be checked and regenerated.

## Scope

**In scope**

- Chat tools: `query_dataset`, `make_chart`, `make_table`, `save_output`.
- The five output types in the table below.
- Vega-Lite specs rendered in the browser with vega-embed.
- HTML tables with CSV export.
- The harness reading numeric values itself from passages or query results.
- Saving each output with its claim ids and query ids.
- The Outputs screen listing saved outputs.

**Out of scope**

- Stale markers and the regenerate button: [F35](f35-output-staleness-derived.md).
- Saving chat answers as derived notes: [F35](f35-output-staleness-derived.md).
- Dataset registration and the DuckDB Analyst worker: [F26](../m06-async-research/f26-datasets-analyst.md).
- Number cross-checks in reports: [F15](../m03-ledger-verification/f15-number-cross-check.md). This feature reuses that approach.

## Design notes

- **Outputs without new research** (Design: Project chat):

  | Output | Built from | Format |
  | --- | --- | --- |
  | Charts | Numeric claims or dataset queries | Vega-Lite spec, rendered in the browser |
  | Tables and comparisons | Claims grouped by entity or question | HTML table, CSV export |
  | Summaries and briefings | The ledger, scoped to a question, cluster or audience | Markdown with citations |
  | Timelines | Dated claims | Vega-Lite or HTML list |
  | Contradiction digests | Contradicts edges | Side-by-side Markdown |

- Every output is saved with the claim and query ids it was built from, so it can be regenerated. (Design: Project chat › Outputs)
- Numbers in charts come from passages or queries, never from the model's prose. The harness reads the values itself. (Design: Project chat › Outputs)
- `query_dataset` runs DuckDB over registered datasets in an Analyst worker. Datasets are never pasted into prompts. (Design: Project chat › How it works; Knowledge store)
- Dataset ingestion failures show on the dataset and on any chart built from it. (Design: Source quality › Your own data)
- Charts use Vega-Lite specs with vega-embed. "Models write the JSON reliably; specs are saved and portable." (Design: Tech stack)
- Outputs screen: "Saved charts, tables and summaries, with stale markers and a regenerate button." (Design: Web interface › Screens)
- Editors save outputs; viewers read them. (Design: Teams and permissions › Roles)

## Implementation sketch

Suggestions, not decisions:

1. Table: `outputs (project_id, kind, title, spec_json or body_md, built_from_claims, built_from_queries, data_snapshot, created_by, created_at, stale)`. Add `stale` now; F35 sets it.
2. `make_chart` takes a Vega-Lite spec without inline data, plus a data source: a list of claim ids or a query id. The harness fills `data.values` from claim structured fields or the stored query result. The model never writes the numbers.
3. For claim-sourced numbers, reuse F15's re-read so each value matches its passage.
4. `query_dataset` sends SQL to the Analyst worker (F26). Store the SQL, dataset version and result rows under a query id.
5. `make_table` groups claims by entity or question. Render with askama, plus `GET /outputs/{id}.csv`.
6. Summaries and digests are Markdown with passage citations, run through the F14 verifier.
7. Vendor a pinned vega-embed bundle. Load it only on pages that show charts.
8. Show dataset check failures as a warning on any chart built from that dataset.

## Acceptance criteria

- [ ] A chart from a dataset query and one from numeric claims both match their sources
- [ ] A chart spec whose model-written numbers differ from the source is corrected or rejected, because the harness fills the values.
- [ ] Each saved output lists the claim ids and query ids it was built from.
- [ ] A table exports as CSV with the same rows the HTML shows.
- [ ] A summary's citations pass the turn-end verifier.
- [ ] A chart built from a dataset with failed ingestion checks shows the warning.

## Open questions

- Vega-Lite specs can include transforms that compute new values. Should transforms be allowed, or should all computation happen in DuckDB, where it's traceable?
- What happens when numeric claims use different units? There's no unit-normalization rule in the design.
- Contradiction digests need F28's contradicts edges, but F28 isn't a dependency of F34. Either build digests after F28, or move that output type to F33 or F35.
