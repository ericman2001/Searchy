# F45 — Video transcript ingestion

**Milestone:** M10 Domain profiles and media · **Status:** ⬜ Not started · **Depends on:** [F11](../m02-local-knowledge/f11-notes-ingestion.md), [F17](../m04-polite-web/f17-fetch-proxy.md) · **Unblocks:** [F44](f44-products-profile.md)

[← Back to plan](../PLAN.md)

## Goal

A lot of product testing and expert explanation lives in video, not text. This feature ingests video as timestamped transcripts, so passages from a video can be searched and cited like any other source. Tapping a citation opens the video at the exact moment.

## Scope

**In scope**

- Fetching transcripts for a video URL through the fetch proxy.
- Chunking transcripts into passages that carry start and end timestamps.
- Indexing these passages in the normal passage store.
- Citations that link to the video at the cited time, plus the archived transcript.
- Marking sources as video in the source-type classifier.

**Out of scope**

- How products weighs video sources (observation vs. opinion): [F44](f44-products-profile.md).
- Making video search part of thorough depth's source list: [F25](../m06-async-research/f25-fan-out-budget.md) / depth preset config.
- Speech-to-text for videos with no transcript. See Open questions.

## Design notes

- Video is ingested as transcripts with timestamp citations. (Design: Worked examples › Robot vacuum comparison)
- The thorough depth adds full-text academic sources and video transcripts to the sources searched. (Design: Depth presets)
- The products profile prefers test-site archives, video transcripts and owner forums. (Design: Domain profiles)
- An observation in a sponsored video still counts, for example "it got stuck on this rug at 4:12." (Design: Worked examples)
- Only the Fetcher touches the network. The Reader works offline on what it archived. (Design: Architecture)
- The fetch proxy prefers APIs and bulk dumps over scraping, never bypasses paywalls or logins, and records paywalled sources as metadata only. (Design: Being a good netizen)
- Every fetched page is archived as WARC, so reruns and verification never refetch. (Design: Being a good netizen)
- Passages are chunked by document structure, not fixed token windows. (Design: Knowledge store › Retrieval)

## Implementation sketch

These are suggestions, not decisions.

- Fetcher: given a video URL, fetch the platform's published captions or transcript where an API or caption track exists. Archive the response in WARC like any page. Respect the same robots, rate and login rules.
- Reader: parse caption formats (WebVTT, SRT, platform JSON) into `(start, end, text)` cues. Group cues into passages at natural boundaries: pauses, chapter markers or sentence ends. Target paragraph-sized passages.
- Passage metadata: `media_start_ms`, `media_end_ms`, `media_url`, and `caption_source` (creator-provided vs. auto-generated).
- Citation renderer: if a passage has `media_start_ms`, link to the video at that time (for example `?t=` for platforms that support it) and also show the archived transcript text.
- Steps: caption fetch in Fetcher → cue parser in Reader → timed chunking → passage fields + migration → citation link rendering → source-type "video."

## Acceptance criteria

- [ ] A citation opens the video at the cited time
- [ ] A WebVTT fixture is parsed into passages whose timestamps match the source cues.
- [ ] Video passages appear in hybrid retrieval results alongside text passages.
- [ ] Rerunning verification on a video citation uses the archived transcript without refetching.
- [ ] A video with no available transcript is recorded as metadata only, with a clear reason.
- [ ] Auto-generated captions are labeled as such on the passage.

## Open questions

- Should Searchy transcribe videos without captions using a local speech model in a worker? The design doesn't mention it. It would be costly, but useful for owner reviews.
- Which platforms are supported first? Some may not expose captions without scraping, which the netizen rules forbid.
- Should auto-generated captions get lower specificity, since they often garble numbers? That matters for F15's number cross-check.
- Dependencies are light (F11, F17), so this could move much earlier in the plan if products work starts sooner (noted in PLAN.md).
