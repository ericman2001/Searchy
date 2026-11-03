# F19 — SearXNG integration

**Milestone:** M4 The web, politely · **Status:** ⬜ Not started · **Depends on:** [F12](../m02-local-knowledge/f12-quick-depth-e2e.md), [F17](f17-fetch-proxy.md) · **Unblocks:** —

[← Back to plan](../PLAN.md)

## Goal

Give research runs a way to search the web. All web search goes through a SearXNG instance, either one the user already runs or a bundled container. SearXNG spreads queries across engines, and the Fetcher then retrieves the results politely.

## Scope

**In scope**

- A SearXNG client that sends queries through the Fetcher and parses the JSON results.
- Settings for an external SearXNG URL, or turning on the bundled SearXNG container.
- A bundled SearXNG container in the Quadlet units and the compose file.
- Wiring web search into the gather step, after local corpora.
- A connectivity test in Settings.

**Out of scope**

- The planner that decides which sub-questions go to the web: [F24](../m06-async-research/f24-planner-checkpoint.md).
- Fan-out across parallel workers: [F25](../m06-async-research/f25-fan-out-budget.md).
- Source scoring of results: [F21](../m05-source-quality/f21-source-classifier-trust.md).
- Academic full-text sources and video transcripts (thorough depth): [F43](../m10-profiles-media/f43-science-health.md), [F45](../m10-profiles-media/f45-video-transcripts.md).

## Design notes

- Search goes through your SearXNG instance, which spreads queries across engines. (Design: Being a good netizen › Fetch proxy)
- First run: point the hub at a SearXNG instance, or use the bundled SearXNG container. Notes are uploaded later in the Library (D6). (Design: Deployment › First run)
- Research pipeline, Gather step: approved sub-questions fan out to workers. Each searches local corpora first, then the web through the fetch proxy. (Design: Research pipeline)
- Depth presets, "Sources searched" (Design: Research pipeline › Depth presets):

| Quick | Standard | Thorough |
| --- | --- | --- |
| Local corpora | Adds the web | Adds full-text academic sources and video transcripts |

- Settings holds model endpoints, API keys, the domain trust list, budgets and local corpora status. SearXNG config belongs there too. (Design: Web interface › Screens)
- Only the Fetcher has network, so SearXNG queries must go through it. (Design: Architecture)

## Implementation sketch

Suggestions only.

1. Add a `search` module with `WebSearch::query(q, opts) -> Vec<SearchHit { url, title, snippet, engine }>`, calling `GET {searxng}/search?q=...&format=json`.
2. Route the call through the Fetcher, but exempt the user's own SearXNG host from robots.txt. Keep a rate limit on it anyway.
3. Bundled option: add a `searxng.container` Quadlet and a compose service on the proxy network, with a minimal `settings.yml` that enables the JSON format.
4. Settings page: an external URL field, a "use bundled" toggle, and a "test search" button.
5. In the gather step, after local retrieval, run web search when the depth is standard or higher. Pass the top N hits to the Fetcher, then to the reader. Dedupe against URLs already in the store.
6. Record which engine returned each hit in the trace.

## Acceptance criteria

- [ ] Standard depth pulls web sources
- [ ] Settings can point at an external SearXNG URL, and the "test search" button reports success or a clear error.
- [ ] The bundled SearXNG container starts from both the Quadlet units and the compose file.
- [ ] Quick depth makes no SearXNG calls.
- [ ] Hits for URLs already archived are not refetched.
- [ ] SearXNG traffic leaves only through the Fetcher container.

## Open questions

- Standard depth as a full preset arrives with the planner in [F24](../m06-async-research/f24-planner-checkpoint.md). Until then, this feature needs a minimal "standard" path on top of [F12](../m02-local-knowledge/f12-quick-depth-e2e.md) to meet its done-when.
- How many results per sub-question to fetch. The design says "fixed sources per sub-question" for standard depth but gives no number.
- First-run wiring is listed as a gap (first-run wizard) in [PLAN.md](../PLAN.md#design-items-with-no-feature-yet).
