# F9 — Passage store and hybrid retrieval

**Milestone:** M2 Local knowledge · **Status:** ⬜ Not started · **Depends on:** [F8](f08-model-routing.md) · **Unblocks:** [F10](f10-kiwix-corpus-manager.md), [F11](f11-notes-ingestion.md), [F12](f12-quick-depth-e2e.md)

[← Back to plan](../PLAN.md)

## Goal
Build the store that every citation in Searchy points at: documents split into passages, each with a stable id, searchable by keyword and by meaning. Reports cite passage ids, never raw URLs, so this is the foundation for claim-level verification. Embeddings run locally, so private notes never leave the machine.

## Scope
**In scope**
- `sources` and `passages` tables with stable passage ids and character spans back into the source text.
- Structure-aware chunking that follows headings and paragraphs, not fixed token windows.
- A `tantivy` BM25 index over passages.
- `sqlite-vec` embeddings produced by the `embeddings` role from F8.
- Hybrid retrieval: BM25 plus vector search, merged with reciprocal rank fusion (RRF).
- Retrieval filters for project, source set and the `local_only` and private-ownership flags.
- An ingestion API that F10, F11 and later fetchers call: `ingest(document) -> passages`.

**Out of scope**
- Specific inputs: Kiwix → [F10](f10-kiwix-corpus-manager.md), notes and PDFs → [F11](f11-notes-ingestion.md), web pages → [F17](../m04-polite-web/f17-fetch-proxy.md)/[F18](../m04-polite-web/f18-warc-archive.md), video → [F45](../m10-profiles-media/f45-video-transcripts.md).
- Claims and the ledger → [F13](../m03-ledger-verification/f13-claim-ledger.md).
- Source scoring → [F21](../m05-source-quality/f21-source-classifier-trust.md).
- Clustering → [F28](../m07-idea-clouds-export/f28-claim-graph-clustering.md).

## Design notes
**Retrieval** (Design: Knowledge store › Retrieval)
- Hybrid search: BM25 keyword search (`tantivy`) plus vector search, merged with reciprocal rank fusion.
- Passages are chunked by document structure (headings, paragraphs), not fixed token windows.
- Embeddings come from a local model through LM Studio or `fastembed-rs`, so private notes never leave the machine.

**Sharing** (Design: Knowledge store › Organization): sources are shared across projects with one archive copy. Trust overrides and notes are per project.

**Privacy** (Design: Teams and permissions › Sharing and privacy): uploaded documents and datasets belong to a person (D6) and are private by default. A personal source that isn't shared never appears in another member's citations. Retrieval must enforce this, not just the UI.

**Tech** (Design: Tech stack): `tantivy` for BM25. `sqlite-vec` for vectors, keeping them in the same database file, with LanceDB as a fallback if it outgrows that. Embeddings via LM Studio or `fastembed-rs`. Crate maturity changes quickly, so check each crate when starting.

## Implementation sketch
Suggestions only.
- Crate `searchy-store`.
- Tables:
  - `sources(id, kind, uri, title, owner_user_id NULL, local_only, content_hash, fetched_at, meta_json)`
  - `passages(id, source_id, ord, heading_path, start, end, text, embedding_model)`
  - `vec_passages` (a `sqlite-vec` virtual table)
  - `project_sources(project_id, source_id, shared_by)`
- Passage ids: stable and content-addressed (for example, a hash of source id plus span plus text), so re-ingesting unchanged text keeps the same ids and old citations still resolve.
- Chunker: parse a Markdown or HTML outline. Emit one passage per paragraph group under a heading path. Split very long paragraphs at sentence boundaries, and merge tiny ones. Keep `heading_path` for display.
- Index tantivy fields `passage_id`, `text`, `heading_path`, `source_kind`, `project_ids` and `owner`. Keep the index directory on `searchy-data`.
- RRF: `score = Σ 1/(k + rank_i)` with `k ≈ 60`. Fetch about 50 from each side and return the top N.
- Record the embedding model per passage, so a model change can trigger re-embedding.
- Retrieval signature: `search(query, Scope { project, source_ids?, viewer_user_id }, n) -> Vec<Hit { passage_id, score, bm25_rank, vec_rank }>`.

## Acceptance criteria
- [ ] Query over a test corpus returns ranked passages with ids
- [ ] A query matching only by keyword and a query matching only by meaning both find the right passage. The fused results rank hits found by both methods highest.
- [ ] Re-ingesting an unchanged document produces identical passage ids.
- [ ] A Markdown document with headings chunks along its headings, and each passage records its heading path.
- [ ] A private source owned by user A that isn't shared to the project never appears in user B's results.
- [ ] Embeddings are produced by the `embeddings` role, and no embedding call goes to a cloud endpoint for local-only sources.

## Open questions

- Embeddings must never fall back to a different model, because vectors from different models (or quantizations) aren't comparable. If the embedding endpoint is down, queue the work. Record the embedding model id with each vector, and re-embed everything when it changes. See [ENVIRONMENT.example.md](../ENVIRONMENT.example.md).
- `sqlite-vec` performance at the scale of a full Wikipedia corpus is untested. Kiwix ([F10](f10-kiwix-corpus-manager.md)) might need a separate index, or lazy embedding of only retrieved articles.
- Passage-id stability when a source changes slightly: old ids should still resolve to the archived text. This interacts with WARC in [F18](../m04-polite-web/f18-warc-archive.md).
- Whether to keep one tantivy index per instance with filter fields, or one per project.
