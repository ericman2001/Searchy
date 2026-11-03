# F30 — Export, Zotero and backup role

**Milestone:** M7 Idea clouds and export · **Status:** ⬜ Not started · **Depends on:** [F16](../m03-ledger-verification/f16-report-page.md) · **Unblocks:** —

[← Back to plan](../PLAN.md)

## Goal

Get research out of Searchy and bring existing libraries in. Reports export as Markdown with citations. Bibliographies export as BibTeX. Zotero libraries go both ways. A `backup` role writes a consistent snapshot of everything durable, so the data volume can be restored.

## Scope

**In scope**

- Markdown export of a report, keeping citations resolvable.
- BibTeX export of a report's or project's sources.
- Zotero import and export.
- BibTeX import of sources with their metadata.
- The `backup` role: a consistent SQLite snapshot plus the WARC archive, written to a target folder.

**Out of scope**

- Exporting chat outputs and their CSVs: [F34](../m08-project-chat/f34-outputs.md). Tables there get their own CSV export.
- The WARC archive itself: [F18](../m04-polite-web/f18-warc-archive.md).
- Scheduling backups. The design only defines the role. A systemd timer or Quadlet can call it (see [F3](../m01-deployable-shell/f03-installer-quadlets.md)).

## Design notes

- BibTeX or Zotero library: "Imported as sources with their metadata." (Design: Knowledge store › Bringing in your own material)
- Imported material is your own trust tier, labeled "your notes" in citations. Whether imported bibliographic entries count as "your notes" or as ordinary sources is not stated. See Open questions. (Design: Knowledge store)
- Sources are shared across projects, with one archive copy. Trust overrides and notes are per project. (Design: Knowledge store › Organization)
- The writer cites passage ids, never raw URLs. Export must turn passage ids into readable references. (Design: Research pipeline step 6)
- Every fetched page is archived as WARC, so citations keep working after the original disappears. (Design: Knowledge store › Retrieval)
- "A `backup` role writes a consistent SQLite snapshot plus the archive to a target folder." (Design: Deployment › Upgrades and operations)
- `searchy-data` holds SQLite, indexes and the WARC archive; it's the volume to back up. `searchy-corpora` is separate and skipped in backups. (Design: Deployment › Storage)
- The same image runs different roles by first argument (`hub`, `fetcher`, `reader`, `analyst`); `backup` is another. (Design: Deployment › One image, several roles)
- SQLite runs in WAL mode. (Design: Teams and permissions › Accounts)
- Personal sources that aren't shared never appear in another member's citations. Exports must respect this. (Design: Teams and permissions)

## Implementation sketch

Suggestions, not decisions:

1. Markdown export: render the report with numbered footnotes. Each footnote holds the source title, author, date, URL or DOI, a quoted passage, and a note that an archived copy exists. Add the confidence badges as plain text.
2. BibTeX export: map source metadata to entry types (`@article`, `@misc`, `@online`). Generate stable citation keys from the source id.
3. BibTeX import: check maturity of Rust BibTeX parsers (e.g. `biblatex`) when the feature starts. Dedupe against existing sources by DOI, then by URL.
4. Zotero: the simplest round trip is file-based (BibTeX or CSL-JSON, which Zotero imports and exports). The Zotero web API is an option, but it adds a network dependency and a key. Decide at implementation time.
5. Backup role: `searchy backup --target /backup`. Use SQLite's online backup API or `VACUUM INTO` for a consistent snapshot. Copy the WARC files and the tantivy index, or document that the index rebuilds from the database. Write a manifest with a timestamp and schema version.
6. Add export buttons on the Report page and a project-level bibliography export.

## Acceptance criteria

- [ ] Report and bibliography open cleanly in Zotero
- [ ] An exported Markdown report renders with every citation resolved to a readable reference. No bare passage ids remain.
- [ ] Importing a BibTeX file with a duplicate DOI doesn't create a duplicate source.
- [ ] `searchy backup` while the hub is writing produces a snapshot that opens and passes `PRAGMA integrity_check`.
- [ ] Restoring a backup into a fresh `searchy-data` volume brings back projects, reports and archived pages.
- [ ] An export by one member never includes another member's unshared personal sources.

## Open questions

- Are imported BibTeX and Zotero entries "your notes" tier or ordinary sources? They describe outside works, so ordinary sources seem more accurate.
- Should the backup include the tantivy index and embeddings, or rebuild them on restore? That trades backup size against restore time.
- Export of saved outputs (charts, tables) is not covered here or in F34 beyond "CSV export." Decide whether report export should embed outputs.
