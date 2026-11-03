# F18 — WARC archiving and archive viewer

**Milestone:** M4 The web, politely · **Status:** ⬜ Not started · **Depends on:** [F17](f17-fetch-proxy.md) · **Unblocks:** —

[← Back to plan](../PLAN.md)

## Goal

Store every page the Fetcher retrieves as a WARC record, and let users open the archived copy from any citation. Citations keep working after the original page changes or disappears. Reruns and verification read the archive instead of refetching.

## Scope

**In scope**

- Writing a WARC request/response record for every live fetch.
- Linking archive records to sources and passages, so a citation can resolve to its archived copy.
- An archive viewer page in the hub that renders the stored copy safely.
- Serving reruns and verification from the archive instead of the network.

**Out of scope**

- Saving pages from a bookmarklet or share sheet. This is listed as a gap in [PLAN.md](../PLAN.md#design-items-with-no-feature-yet).
- Backups of the archive: [F30](../m07-idea-clouds-export/f30-export-backup.md).
- Video archiving: [F45](../m10-profiles-media/f45-video-transcripts.md).

## Design notes

- Every fetched page is archived as WARC, so citations keep working after the original changes or disappears. (Design: Knowledge store › Retrieval)
- Every fetched page is stored as WARC, which also means reruns and verification never refetch. (Design: Being a good netizen › Archiving)
- Sources are shared across projects with one archive copy. Trust overrides and notes are per project. (Design: Knowledge store › Organization)
- The Sources screen shows each source's archive copy. (Design: Web interface › Screens)
- The Report screen: tap a citation to see the exact passage. (Design: Web interface › Screens)
- Storage: the `searchy-data` volume holds SQLite, indexes and the WARC archive. It is the volume to back up. (Design: Deployment › Storage)
- Suggested crate: `warc`. (Design: Tech stack)

## Implementation sketch

Suggestions only.

1. Have the Fetcher write each response into a rolling WARC file under `/data/warc/`, for example one file per day or per size cap, gzip-compressed per record.
2. Add a `archive_records` table: `id, url, warc_file, offset, length, fetched_at, content_type, sha256`. Link `sources.archive_record_id` to it.
3. Make the fetch path check the archive first when the request comes from a rerun or verification. Only go to the network for new URLs or explicit refreshes.
4. Archive viewer: `GET /archive/{record_id}` reads the record at its offset and serves the body.
   - Serve HTML in a sandboxed iframe with a strict CSP (no scripts, no network), or as extracted text.
   - Show a banner with the original URL and the capture date.
5. Passage deep links: `/archive/{record_id}#passage-{id}` highlights the cited span when the passage offsets are known.

## Acceptance criteria

- [ ] Citation opens the archived copy after the origin is offline
- [ ] Every successful live fetch produces exactly one WARC response record, and its offset is recorded in SQLite.
- [ ] Archived HTML renders without running scripts or making network requests.
- [ ] A rerun of an existing question makes no network requests for already-archived URLs.
- [ ] Two projects citing the same URL share one archive record.
- [ ] The archive viewer shows the original URL and the capture date.

## Open questions

- How refreshes are triggered for volatile pages (news, prices) when the archive is meant to prevent refetching. Decay rules arrive with profiles in [F41](../m10-profiles-media/f41-profile-system.md).
- Whether to store paywalled-source metadata as a WARC `metadata` record or only in SQLite.
