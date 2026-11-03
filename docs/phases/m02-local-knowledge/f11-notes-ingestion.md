# F11 — Notes and PDF ingestion

**Milestone:** M2 Local knowledge · **Status:** ⬜ Not started · **Depends on:** [F9](f09-passage-store.md) · **Unblocks:** [F12](f12-quick-depth-e2e.md), [F45](../m10-profiles-media/f45-video-transcripts.md)

[← Back to plan](../PLAN.md)

## Goal
Make your own notes and PDFs first-class, searchable sources next to Wikipedia and the web. You upload them in the Library, and they're indexed into the database store. Re-uploading an edited note replaces it and re-indexes it. PDFs are extracted in a no-network worker so their contents can't leak.

Per decision D6 in PLAN.md, there are no host folder mounts or watched folders.

## Scope
**In scope**
- Uploading Markdown notes and PDFs from the Library screen, one at a time or several at once, with ingestion status shown there.
- Re-uploading a document with the same name (or choosing "Replace" on it) replaces it and re-indexes it within a minute. Deleting a document removes it from search.
- Markdown links, front matter and tags kept as metadata.
- PDF text extraction in a reader worker with no network.
- Per-person ownership: documents are private by default, and sharing one to a project is explicit.
- Optional collections: named groups of documents, so sharing and settings can apply to many at once.
- The local-only flag on a document or collection.
- A user-set reliability per document or collection (stored here; scoring uses it later).

**Out of scope**
- Datasets (CSV, Parquet, SQLite) → [F26](../m06-async-research/f26-datasets-analyst.md).
- BibTeX and Zotero import → [F30](../m07-idea-clouds-export/f30-export-backup.md).
- Saving web pages with a bookmarklet or share sheet (listed as a gap in PLAN.md).
- OCR for scanned PDFs (listed as a gap in PLAN.md, possibly folded in here).
- Editing notes inside Searchy. Notes are edited elsewhere and re-uploaded.
- Using note reliability in claim confidence → [F23](../m05-source-quality/f23-claim-confidence.md), [F37](../m09-review/f37-reliability-judges-labels.md).

## Design notes
**Inputs** (Design: Knowledge store › Bringing in your own material)

| Input | How it's ingested |
| --- | --- |
| Markdown notes | Uploaded in the Library, singly or several at once; re-uploading replaces and re-indexes; links and tags kept as metadata |
| PDFs | Text extraction in a no-network worker; scanned PDFs get OCR |

- Your material is its own trust tier, labeled "your notes" in citations, so you can see when an answer leans on you rather than the outside world.
- Uploaded notes are their own tier, with a reliability you set per document or collection (Design: Source quality › Your own data).

**Privacy** (Design: Teams and permissions › Sharing and privacy)
- Uploaded documents belong to a person and are private by default.
- Adding one to a project is an explicit share. Project members see only what was shared.
- A personal source that isn't shared never appears in another member's citations.
- A document or collection can be marked local-only, so its contents never go to a cloud model. The model gateway enforces this (D4; Design: Model routing › Rules).

**Isolation** (Design: Architecture): only the Fetcher touches the network. The Reader works offline, so a malicious PDF can't exfiltrate notes. Model calls from workers go through the hub's model gateway over the internal network (D4).

**Storage** (Design: Deployment › Storage): uploaded documents live in `searchy-data` with everything else. There are no host folder mounts.

**PDF tools** (Design: Tech stack): `pdf-extract`, or Poppler in the reader image. Poppler is more robust for odd PDFs.

## Implementation sketch
Suggestions only.
- Upload endpoint: multipart upload from the Library page (an htmx form, with `multiple` file selection). Store the original bytes under `/data/documents/<owner>/<doc_id>` with a content hash.
- Replace: a re-upload matched by the owner's filename (or an explicit "Replace" action) with a new hash creates a new version. Re-chunk it through F9 and tombstone the old version's passages rather than hard-deleting them, so old citations still render.
- Each document becomes a source with `kind = "note"` or `"pdf"`, plus `owner_user_id`. `local_only` and `reliability` come from the document, falling back to its collection.
- Markdown metadata: parse YAML front matter, `#tags`, and `[[wikilinks]]` into `meta_json`. Resolve wikilinks against the owner's other documents by title, and store them as source-to-source references for the idea map.
- PDFs: the hub copies the file into a job folder. The launcher starts a `reader` worker with no network and a read-only mount. The worker runs `pdftotext -layout` (Poppler) or `pdf-extract` and writes text plus page spans back. The hub chunks the text, keeping page numbers in the passage metadata for citations.
- Tables: `documents(id, owner_user_id, collection_id, filename, hash, version, local_only, reliability, status, uploaded_at)`, `collections(id, owner_user_id, name, local_only, reliability)`, plus the `project_sources` share rows from F9.
- Library screen: list documents and collections with status (queued, indexing, ok, error), with replace, delete and share-to-project controls.

## Acceptance criteria
- [ ] Re-upload an edited note; it is re-indexed within a minute
- [ ] Deleting a document removes it from search results, and existing citations to it still render from the stored text.
- [ ] Markdown tags and wikilinks show up in the source's metadata.
- [ ] Uploading several files at once ingests all of them, with each one's status shown in the Library.
- [ ] An uploaded PDF is extracted by a reader worker that has no network. The trace shows the worker role and the network mode.
- [ ] A citation to a PDF passage shows its page number.
- [ ] A document that isn't shared to a project never appears in another member's search results for that project.

## Open questions
- Resolved (D6): there are no folder mounts. Each person uploads their own documents.
- Should a whole Obsidian vault be uploadable as a zip, keeping its folder structure as a collection? It's convenient, but zip parsing needs size and path-traversal limits.
- OCR for scanned PDFs is in the design but unassigned (see the PLAN.md gaps). It needs Tesseract or similar in the reader image.
- Markdown is parsed in the hub, which has network access. PDFs never are. Confirm that this split is an acceptable risk.
