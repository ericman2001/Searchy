# F10 — Kiwix container and corpus manager

**Milestone:** M2 Local knowledge · **Status:** ⬜ Not started · **Depends on:** [F6](../m01-deployable-shell/f06-job-queue-launcher.md), [F7](../m01-deployable-shell/f07-progress-notifications.md), [F9](f09-passage-store.md) · **Unblocks:** [F12](f12-quick-depth-e2e.md)

[← Back to plan](../PLAN.md)

## Goal
Give Searchy a local copy of Wikipedia, so general background questions are answered without touching anyone's server. It also adds a corpus manager in Settings that downloads corpora as background jobs and shows their progress in the Inbox. Other corpora can plug into the same manager later.

## Scope
**In scope**
- Running `kiwix-serve` from its own upstream image, as a container next to the hub.
- A corpus manager: list available corpora, start a download as a background job, show progress and status, and delete or update a corpus.
- The default Wikipedia download: Kiwix's no-pictures English build.
- Making Wikipedia searchable through the hub's retrieval, with Wikipedia passages citable by passage id.
- The corpus step of the first-run flow (Design: Deployment › First run, step 3).

**Out of scope**
- The other local corpora (Wikidata, arXiv, OpenAlex snapshot, Stack Exchange, Gutenberg). These are listed as a gap in PLAN.md and suggested as follow-ups to F10.
- PubMed baseline and PMC corpora → [F43](../m10-profiles-media/f43-science-health.md).
- Live web fetching → [F17](../m04-polite-web/f17-fetch-proxy.md).

## Design notes
**Local corpora** (Design: Being a good netizen)
- Wikipedia is a Kiwix ZIM file served by `kiwix-serve` in its own container. It's used for general background, entities and definitions.
- The default build is Kiwix's no-pictures English build: full article text without images. ZIM files contain articles only, so talk pages are never included.
- Corpus sizes and update schedules should be checked at install time, not hard-coded.
- Corpora are optional and installed from Settings.

**Deployment** (Design: Deployment): Kiwix runs as its own upstream image. Corpora live on the `searchy-corpora` volume, kept separate so it can sit on a big, slow disk and be skipped in backups.

**Inbox** (Design: Web interface › Screens; Deployment › First run): downloads run as background jobs with progress in the Inbox.

**Admin only** (Design: Teams and permissions › Roles): admins manage corpora.

## Implementation sketch
Suggestions only.
- Corpus catalog: fetch the Kiwix library catalog (OPDS) at install or browse time, to get the current file name, size and date for the English no-pictures build. Don't hard-code a URL or size.
- Download job: a fetcher-role job writes to `/corpora/wikipedia/<file>.zim.part`, verifies the checksum, then renames the file. It's resumable with HTTP Range and reports bytes over the job progress channel (SSE from F7).
- The launcher (F6) gains one more fixed menu item: start or restart the `kiwix` container with `/corpora` mounted read-only, on an internal network the hub can reach.
- Search integration options. Choose one after measuring:
  1. Query `kiwix-serve`'s full-text search at request time. Fetch the top articles, chunk them through F9, and cache those passages. Embeddings are lazy, created only for articles that are actually retrieved.
  2. Bulk-index the whole ZIM into tantivy and `sqlite-vec`. Embedding millions of passages is heavy.
  
  Option 1 is the likely starting point.
- Store sources as `kind = "corpus:wikipedia"`, with a URI that names the article and the ZIM version, so a citation records which snapshot it came from.
- Table: `corpora(id, name, kind, file, version, size_bytes, status, installed_at, job_id)`.

## Acceptance criteria
- [ ] Wikipedia ZIM searchable through the hub
- [ ] Starting a download from Settings creates a background job, and its progress appears in the Inbox and survives a page reload.
- [ ] An interrupted download resumes instead of restarting.
- [ ] The `kiwix` container is started through the launcher, with the corpus mounted read-only.
- [ ] A search result from Wikipedia has a passage id and records the ZIM version.
- [ ] Only admins can install or remove corpora.

## Open questions
- On-demand passage creation (option 1) versus a full index. The choice affects retrieval quality and the disk and CPU budget on small hosts such as a Pi 5.
- Update cadence: does a new ZIM release become a new version that old citations still point at, or does it replace the old one?
- Resolved (D5): the download is a queued fetcher-role job started by the launcher, like every other Fetcher job.
- Bulk dumps still need politeness rules from [F17](../m04-polite-web/f17-fetch-proxy.md), which comes later in M4. F10 may ship a minimal fetcher-role downloader (one connection, resumable, honest User-Agent) that F17 later absorbs.
