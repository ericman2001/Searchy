# F7 — SSE progress, Inbox, Web Push and PWA

**Milestone:** M1 Deployable shell · **Status:** ⬜ Not started · **Depends on:** [F6](f06-job-queue-launcher.md) · **Unblocks:** [F10](../m02-local-knowledge/f10-kiwix-corpus-manager.md), [F24](../m06-async-research/f24-planner-checkpoint.md), [H9](../mh-pipeline-core/h09-turn-end-validation.md), [H14](../mh-pipeline-core/h14-interruption-branching-replay.md)

[← Back to plan](../PLAN.md)

## Goal

Make the asynchronous experience work from a phone. Run pages show live progress. The Inbox collects everything that needs you. A notification arrives when a job finishes, even with the browser closed. Over plain HTTP that's ntfy. Over HTTPS, Web Push works and the hub installs to the home screen like an app (D3).

## Scope

**In scope**

- An SSE endpoint per run or job that streams progress events. On reconnect, the page re-renders from the database.
- A Run page skeleton with current step, counters and status, driven by SSE.
- The Inbox screen, listing what needs the user. For M1 that means finished and failed jobs, with room for later types.
- Counselor questions from paused run steps (D9). An `AskUser` from a counselor ([H8](../mh-pipeline-core/h08-counselors.md)) becomes an Inbox item with its choices. Answering it resumes that step; other branches of the run keep going meanwhile. F7 builds the item type, the answer form and the resume signal; H8 raises the questions.
- ntfy notifications, configurable per user. This is the path that works over plain HTTP.
- Web Push when served over HTTPS: VAPID keys, subscription management per device, and sending a push on job completion or failure.
- A PWA manifest, icons and a minimal service worker (needed to receive push), active only in a secure context.
- Feature detection: the notification settings page offers Web Push and "Add to home screen" only when `window.isSecureContext` is true, and otherwise points to ntfy (D3).

**Out of scope**

- Plan-approval notifications and approving from a push → [F24](../m06-async-research/f24-planner-checkpoint.md). The Inbox item type should be ready for it.
- Real run metrics (sources fetched, claims found, cost) → filled in by [F12](../m02-local-knowledge/f12-quick-depth-e2e.md), [F13](../m03-ledger-verification/f13-claim-ledger.md) and [F25](../m06-async-research/f25-fan-out-budget.md).
- Review and watch notifications → [F27](../m06-async-research/f27-watches.md), [F40](../m09-review/f40-verdict-card.md).
- Chat token streaming → [F31](../m08-project-chat/f31-project-chat-loop.md). It reuses the same SSE transport.
- Corpus download progress → [F10](../m02-local-knowledge/f10-kiwix-corpus-manager.md).
- Mention notifications for comments. This is a gap item in PLAN.md.

## Design notes

**Async behavior** (Design: Web interface › Async behavior)

- All state lives in SQLite on the host, so closing the tab or losing signal changes nothing.
- Progress streams over Server-Sent Events. On reconnect, the page re-renders from the database.
- Web Push (the browser's push API, no app store) notifies you when a plan needs approval or a run finishes. ntfy is a simpler fallback.
- HTTPS is optional (D3). Browsers allow service workers, Web Push and PWA install only in a secure context, which `http://<lan-ip>` isn't. Over plain HTTP, ntfy is the notification path.
- The hub is installable as a PWA, so it opens like an app from the home screen.

**Screens** (Design: Web interface › Screens)

| Screen | What you do there |
| --- | --- |
| Inbox | See what needs you: plans awaiting approval, finished runs and reviews, failed jobs, watch updates, and questions from paused run steps (D9) |
| Run | Live progress over SSE: current step, sources fetched, claims found, cost so far |

**UI** (Design: Web interface): server-rendered HTML with htmx, phone-first, works on a slow connection, no JS framework. (Design: Tech stack): htmx + SSE; chat streaming uses the same transport.

## Implementation sketch

Suggestions only:

- **Events**: a `run_events(id, run_id, job_id, kind, payload_json, at)` table, plus an in-process `tokio::sync::broadcast` per run. Workers report progress through a narrow hub endpoint on the internal model network, which they can reach (D4), or through their job folder. The hub writes the event, then broadcasts it.
- **SSE**: `axum::response::sse` with the htmx SSE extension (`hx-ext="sse"`, `sse-swap`). Send rendered HTML fragments, so there's no client logic. Use `Last-Event-ID` to replay missed events, or simply re-render the full partial on connect.
- **Inbox**: `inbox_items(id, user_id, kind, project_id, ref_type, ref_id, created_at, read_at, dismissed_at)`.
  - Kinds: `job_finished`, `job_failed`, `step_question`. Reserve `plan_awaiting_approval`, `review_finished`, `watch_update`.
  - `step_question` stores the question, its choices and the paused step id. Answering writes the answer, marks the item done and signals the run to resume that step.
  - Inbox items respect project membership.
- **Web Push**:
  - Generate VAPID keys on first start and store them in `/data`.
  - `push_subscriptions(id, user_id, endpoint, p256dh, auth, device_label, created_at, last_ok_at)`.
  - Send with a Rust web-push crate (check maturity) and drop subscriptions that return 410.
  - Push payloads stay minimal (title and a link), since content passes through the browser vendor's push service.
- **ntfy**: per-user `ntfy_url` plus topic. Send a POST on the same events.
- **PWA**: `manifest.webmanifest` (name, icons, `display: standalone`, `start_url: /inbox`) and `sw.js` with `push` and `notificationclick` handlers. Keep the service worker small, and skip offline caching for now.
- **iOS**: Web Push only works for home-screen-installed PWAs (iOS 16.4+). Add a hint on the subscribe page.
- **Settings**: a per-device "Enable notifications" button and a test-push button.

## Acceptance criteria

- [ ] Phone gets a push when a dummy job finishes
  - Over plain HTTP this is an ntfy notification. Over HTTPS it's Web Push.
- [ ] The Run page updates live during a dummy job. After closing and reopening it mid-job, it shows the correct current state.
- [ ] A failed dummy job appears in the Inbox and sends a push.
- [ ] Dismissing an Inbox item removes it for that user only.
- [ ] Over HTTPS, the hub can be added to a phone home screen and opens standalone. Over plain HTTP, the settings page hides Web Push and install, and offers ntfy instead.
- [ ] A dummy step that raises a question pauses, the question appears in the Inbox, and answering it resumes the step while a sibling step keeps running.
- [ ] With ntfy configured and push disabled, the ntfy topic receives the notification.

## Open questions

- Resolved (D3): Web Push and PWA install need HTTPS, which is optional. ntfy covers plain HTTP.
- How do workers report progress? Under D4 they can reach the hub on the internal model network, so a narrow progress endpoint there is the likely channel. Job-folder files remain a fallback.
- Mention notifications for comments: gap item in PLAN.md.
