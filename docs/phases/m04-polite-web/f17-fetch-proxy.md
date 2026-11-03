# F17 — Fetch proxy worker

**Milestone:** M4 The web, politely · **Status:** ⬜ Not started · **Depends on:** [F6](../m01-deployable-shell/f06-job-queue-launcher.md), [H8](../mh-pipeline-core/h08-counselors.md) · **Unblocks:** [F18](f18-warc-archive.md), [F19](f19-searxng.md), [F20](f20-injection-counselor.md), [F21](../m05-source-quality/f21-source-classifier-trust.md), [F36](../m09-review/f36-provenance-tiers.md), [F38](../m09-review/f38-review-job.md), [F45](../m10-profiles-media/f45-video-transcripts.md)

[← Back to plan](../PLAN.md)

## Goal

Build the Fetcher: the only worker in the system that touches the internet. It is a queued per-job worker (D5), started by the launcher like the other roles. Every live web request goes through it and is polite by construction. It obeys robots.txt, rate-limits per domain, backs off when servers push back, and caches so a page is fetched once across all projects. Because only the Fetcher has network, a malicious page can't make the Reader or Analyst leak your notes.

## Scope

**In scope**

- `searchy fetcher` worker role: a queued per-job worker (D5), launched by the launcher sidecar on the proxy network plus the internal model network.
- Politeness that holds across concurrent Fetcher jobs, since several can run at once.
- robots.txt parsing and enforcement, including `Crawl-delay`.
- Per-domain rate limits and concurrency caps.
- Exponential backoff on 429 and 503.
- Honest User-Agent with the project name and a contact URL.
- Shared HTTP cache with conditional requests (ETag, If-Modified-Since).
- Paywall and login detection: record metadata only, never bypass.
- Preferring APIs and bulk dumps over scraping where a site offers them (a hook for per-site handlers).

**Out of scope**

- WARC archiving of responses: [F18](f18-warc-archive.md).
- Web search: [F19](f19-searxng.md).
- Injection screening of fetched text: [F20](f20-injection-counselor.md).
- Source-type classification: [F21](../m05-source-quality/f21-source-classifier-trust.md).
- Crossref and OpenAlex API clients: [F22](../m05-source-quality/f22-crossref-openalex.md).
- Video transcripts: [F45](../m10-profiles-media/f45-video-transcripts.md).

## Design notes

- Only the Fetcher touches the network. The Reader and Analyst work offline on what it archived and on your own files. (Design: Architecture)
- The launcher picks the network: none for every worker except the Fetcher, which joins the proxy network. (Design: Deployment › Isolation)
- Proxy rules (Design: Being a good netizen › Fetch proxy):
  - Obeys robots.txt and honors `Crawl-delay`.
  - Per-domain rate limits and concurrency caps, with exponential backoff on 429 and 503.
  - An honest User-Agent naming the project with a contact URL.
  - A shared HTTP cache with conditional requests, so a page is fetched once across all projects.
  - Prefers APIs and bulk dumps over scraping.
  - Never bypasses paywalls or logins. A paywalled source is recorded as metadata only.
- Local corpora answer first. The live web is fetched only when needed. Every live fetch is a cost to someone else's server. (Design: Goals; Being a good netizen)
- Classifiers include paywall detection. (Design: Research pipeline › harness stage mapping)
- Domain politeness is one of the counselors. (Design: Research pipeline › harness stage mapping)
- Sources are shared across projects with one archive copy. (Design: Knowledge store › Organization)
- Suggested crates: `reqwest` + `texting_robots` + `governor`. (Design: Tech stack)

## Implementation sketch

Suggestions only. Check crate maturity before committing.

1. Add a `fetch` crate with a `FetchRequest { url, purpose, job_id }` → `FetchResult { status, headers, body_ref, from_cache, paywalled }` API.
2. Transport (D5): the Fetcher is a queued per-job worker. A fetch job's `in/job.json` lists the URLs and purposes; results land in `out/` and the shared cache, where no-network workers read them through their job folders. Other workers never call the Fetcher directly; they ask the hub, which enqueues fetch jobs.
   - Cross-job politeness: per-domain state (next allowed time, in-flight count, backoff until) lives in a hub-side SQLite table. A Fetcher job claims each URL from the hub before requesting it, so limits hold even when several Fetcher jobs run at once. The in-process `governor` limiter (step 4) is then a second line of defense inside one job.
3. robots.txt: fetch it once per host, cache it with a TTL, and evaluate with `texting_robots`. Use `Crawl-delay` as a floor on the per-host interval.
4. Rate limiting: one `governor` limiter per domain, plus a semaphore for the concurrency cap. Defaults come from config, and Settings can override them per domain.
5. Backoff: on 429 or 503, honor `Retry-After` if present, otherwise back off exponentially with jitter, up to a maximum retry count.
6. Cache: a `http_cache` table (url, etag, last_modified, fetched_at, body hash) with bodies on the `searchy-data` volume. Revalidate with conditional requests and treat 304 as a hit.
7. User-Agent: `Searchy/<version> (+<contact URL>)`. The contact URL comes from instance settings.
8. Paywall: simple heuristics (status 401/402/403, login redirects, known paywall markers) set `paywalled = true` and keep metadata only.
9. A `SiteHandler` trait for "prefer the API" cases, starting with none and filled in by later features.

## Acceptance criteria

- [ ] Load test against a local server never exceeds the configured rate
- [ ] A URL disallowed by robots.txt is never requested, and the run trace records why.
- [ ] `Crawl-delay: 5` on a test host spaces requests at least 5 seconds apart.
- [ ] A test server returning 429 with `Retry-After` is retried no sooner than the header says.
- [ ] A second fetch of an unchanged page sends `If-None-Match` or `If-Modified-Since` and is served from cache on 304.
- [ ] A login-walled test page is recorded as metadata only, with no body stored.
- [ ] No other worker role can reach the internet; only the Fetcher container is on the proxy network.
- [ ] Two concurrent Fetcher jobs hitting the same test host together stay within that host's configured rate.

## Open questions

- Resolved (D5): the Fetcher is a queued per-job worker, so F6's launcher menu needs no long-lived service.
- Where the default per-domain limits live (instance settings, profile TOML, or both).
- The "domain politeness" counselor plugs into the counselor hooks from [H8](../mh-pipeline-core/h08-counselors.md).
