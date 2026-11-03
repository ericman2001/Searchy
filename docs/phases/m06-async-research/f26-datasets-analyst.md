# F26 — Datasets and the Analyst worker

**Milestone:** M6 Async research · **Status:** ⬜ Not started · **Depends on:** [F6](../m01-deployable-shell/f06-job-queue-launcher.md), [F8](../m02-local-knowledge/f08-model-routing.md), [F12](../m02-local-knowledge/f12-quick-depth-e2e.md) · **Unblocks:** [F34](../m08-project-chat/f34-outputs.md), [F37](../m09-review/f37-reliability-judges-labels.md), [F47](../m10-profiles-media/f47-remaining-profiles.md)

[← Back to plan](../PLAN.md)

## Goal

Make your own data a first-class source. Upload a CSV, Parquet file or SQLite database, and an Analyst worker queries it in place with DuckDB. Data is never pasted into a prompt. Answers cite the exact query that produced each number.

## Scope

**In scope**

- Registering CSV, Parquet and SQLite files as datasets, with provenance (where the data came from).
- The `analyst` worker role: no network, read-only dataset mounts, DuckDB.
- A query tool the pipeline can call: the model writes SQL, the Analyst runs it, and results come back as a cited query result.
- Dataset ownership and sharing: private to the uploader until added to a project.
- Local-only datasets never sent to cloud models.
- Library screen entries for datasets.

**Out of scope**

- Dataset ingestion checks (schema, null and range checks): [F37](../m09-review/f37-reliability-judges-labels.md).
- Charts and tables built from queries: [F34](../m08-project-chat/f34-outputs.md).
- The `query_dataset` chat tool: [F31](../m08-project-chat/f31-project-chat-loop.md), [F34](../m08-project-chat/f34-outputs.md).

## Design notes

- CSV, Parquet, SQLite: registered as datasets. The Analyst worker queries them with DuckDB and never pastes them into prompts. (Design: Knowledge store › Bringing in your own material)
- The Reader and Analyst work offline on what the Fetcher archived and on your own files. (Design: Architecture)
- The Analyst is a worker role in the single image: `hub`, `fetcher`, `reader` or `analyst`. (Design: Deployment)
- Workers run in read-only containers with tmpfs scratch and no internet route; the launcher mounts only that job's folders. Model calls go through the hub's model gateway on the internal network (D4). (Design: Deployment › Isolation)
- Datasets belong to a person and are private by default. Adding one to a project is an explicit share. (Design: Teams › Sharing and privacy)
- Local-only data stays local. A role that would send a local-only dataset to a cloud model is rerouted to a local model, or the step is skipped and flagged. (Design: Model routing › Rules)
- Datasets record where they came from. (Design: Source quality › Your own data)
- Library screen: upload notes, PDFs and CSVs; see ingestion status and data checks. (Design: Web interface)
- Data analysis is one of the question types. (Design: Research pipeline, Scope)
- Suggested crate: `duckdb`. (Design: Tech stack)
- Technical-docs profile later runs code in an Analyst worker to test claims. (Design: Domain profiles)

## Implementation sketch

Suggestions only.

1. Tables:
   - `datasets (id, owner_id, name, path, format, provenance, local_only, row_count, created_at)`
   - `project_datasets (project_id, dataset_id, shared_by, at)`
   - `dataset_queries (id, dataset_id, sql, result_hash, result_ref, run_id, at)`
2. Upload: Library screen form, streamed to `/data/datasets/{id}/`. Capture a provenance field (URL or free text).
3. Schema introspection with DuckDB at registration. Store column names, types and a few sample rows. Only the schema, not the data, goes to the model (rerouted locally if the dataset is local-only).
4. Analyst job: the launcher mounts the dataset read-only and passes the SQL. DuckDB runs it with a row limit and timeout and writes the result to the job folder.
5. SQL safety: run in read-only mode, allow only `SELECT`/`WITH`, and limit rows and time.
6. Citation: a query result becomes a citable unit with id, SQL text and dataset, like a passage id. The report renders it as "query Q12 on dataset X".
7. Pipeline hook: when the question type is data analysis and the project has datasets, the planner adds an analyst sub-question.

## Acceptance criteria

- [ ] Question about your CSV answered with a cited query
- [ ] CSV, Parquet and SQLite files each register and show their schema in the Library.
- [ ] The Analyst container has no network and mounts only the datasets for its job.
- [ ] A non-SELECT statement is rejected.
- [ ] Tapping a query citation shows the SQL, the dataset and the result.
- [ ] A local-only dataset's schema and results never appear in a cloud-model call in the trace.
- [ ] A dataset not shared to a project is not queryable from that project.

## Open questions

- The done-when needs a question pipeline, but F26 depends only on F6 and F8. Answering a question also needs [F12](../m02-local-knowledge/f12-quick-depth-e2e.md) (or [F24](f24-planner-checkpoint.md) to add an analyst sub-question). Treat F12 as an implicit dependency.
- Upload size limits for large datasets. Host folder mounts were dropped (D6), so large files must be uploaded; consider chunked or resumable uploads.
- Whether dataset query results count toward claim corroboration, and at what tier. The design calls datasets primary sources in the provenance table.
