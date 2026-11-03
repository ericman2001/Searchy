# F6 — Job queue, Runner trait and launcher sidecar

**Milestone:** M1 Deployable shell · **Status:** ⬜ Not started · **Depends on:** [F5](f05-projects-data-model.md) · **Unblocks:** [F7](f07-progress-notifications.md), [F10](../m02-local-knowledge/f10-kiwix-corpus-manager.md), [F17](../m04-polite-web/f17-fetch-proxy.md), [F26](../m06-async-research/f26-datasets-analyst.md)

[← Back to plan](../PLAN.md)

## Goal

Let the hub run work in the background, in short-lived, isolated worker containers, without ever holding the container socket itself. This is what lets research keep running after the browser closes. It is also the main security boundary of the system.

## Scope

**In scope**

- A SQLite-backed job queue: enqueue, claim, heartbeat, complete, fail and cancel.
- A `Runner` trait with `podman`, `docker` and `local` backends.
- A launcher sidecar process (the same binary, a `launcher` role) that mounts the container socket. It serves a fixed request menu over a Unix socket shared with the hub.
- Launcher policy: image pinned by digest, per-job mounts, networks per role, resource limits, and audit logging of every request. Every worker joins only the internal model network, which reaches the hub's model gateway and nothing else (D4). The Fetcher also joins the proxy network.
- All worker roles, including the Fetcher, are queued per-job workers started through this one menu (D5). There are no long-lived worker services.
- Restart recovery: jobs running when the hub died are re-queued.
- A dummy job type for testing.
- Fair scheduling across users (the PLAN.md gap item suggests F6).

**Out of scope**

- Live progress UI and push → [F7](f07-progress-notifications.md).
- Real worker logic: fetcher → [F17](../m04-polite-web/f17-fetch-proxy.md), reader → [F11](../m02-local-knowledge/f11-notes-ingestion.md), analyst → [F26](../m06-async-research/f26-datasets-analyst.md).
- The proxy network setup for the Fetcher → [F17](../m04-polite-web/f17-fetch-proxy.md). The launcher should support it, though.
- The model gateway itself → [F8](../m02-local-knowledge/f08-model-routing.md) and [H2](../mh-pipeline-core/h02-provider-adapters.md). F6 only attaches workers to the internal network it listens on.
- Budgets → [F25](../m06-async-research/f25-fan-out-budget.md).

## Design notes

**Architecture** (Design: Overview; Architecture): the hub is always on. Short-lived worker containers do the research. Volumes hold everything durable. This comes from the Podman hub design: SQLite job queue, sibling worker containers with tmpfs scratch, Quadlet deployment. Only the Fetcher touches the network.

**Launcher sidecar** (Design: Deployment › Isolation)

- A small container is the only thing that mounts the rootless Podman socket.
- It listens on a Unix socket shared with the hub (volume `searchy-launcher` → `/run/searchy-launcher`).
- It accepts a fixed menu: **start** a worker of a given role for a job, **stop** it, report **status**, stream **logs**.
- The launcher chooses the image (pinned by digest), the mounts (only that job's folders), the networks and the resource limits. It rejects anything else and logs every request.
- Networks (D4): every worker joins the internal model network (`Internal=true`, no internet route), where the only reachable service is the hub's model gateway. The Fetcher additionally joins the proxy network (D5).
- A compromised hub can only start approved workers on approved paths. A compromised worker is still inside a read-only container with tmpfs scratch, with no internet route.

**Runner backends** (Design: Deployment › Worker runner backends)

- `podman`: sibling containers via the launcher with the rootless Podman socket. This is the default.
- `docker`: the same launcher, holding the Docker socket.
- `local`: workers as child processes in the hub's container, with no extra isolation. The UI warns that it is less isolated.

**Fair scheduling** (Design: Teams and permissions › Attribution and coordination): the queue schedules fairly across users, so one person's thorough run can't starve everyone else.

**Stack** (Design: Tech stack): Podman and Docker API via `bollard`, one client for both sockets.

## Implementation sketch

Suggestions only:

- **Table**: `jobs(id, kind, role, project_id, run_id, owner_user_id, payload_json, status, attempts, claimed_at, heartbeat_at, container_id, created_at, finished_at, error)`.
  - `status` ∈ `queued | starting | running | succeeded | failed | cancelled`.
- **Scheduler**: a hub task loop. Pick the next job round-robin by `owner_user_id`, then by FIFO within a user. Cap concurrency globally and per role.
- **Runner trait**: `start(JobSpec) -> Handle`, `stop`, `status`, `logs(stream)`. `JobSpec` holds only the role and job id. The launcher derives everything else.
- **Launcher protocol**: newline-delimited JSON over a Unix socket, with a closed enum of request types. Anything that doesn't parse into the enum is rejected and logged. The launcher reads its policy (role → image digest, mounts, network, limits) from its own config, never from the request.
- **Network policy**: a fixed role → networks map in the launcher config, e.g. `reader = ["searchy-models"]`, `fetcher = ["searchy-models", "searchy-proxy"]`. The request can't name networks.
- **Job folders**: `/data/jobs/<job_id>/{in,out}`. The worker gets only that folder. Root is read-only, with tmpfs at `/tmp`.
- **Worker contract**: read `in/job.json`, write `out/result.json`, and set the exit code. The hub ingests the results.
- **Recovery**: on hub startup, check every `running` or `starting` job against launcher `status`. Re-queue lost jobs, and fail them after N attempts. Use heartbeats to detect stuck containers.
- **Local backend**: `tokio::process::Command` running the same binary with the role argument. Show a banner in Settings when it's active.
- Add the launcher unit to the [F3](f03-installer-quadlets.md) Quadlets and compose file.

## Acceptance criteria

- [ ] Dummy job runs in a sibling container; the hub has no socket; an off-menu launcher request is rejected and logged; hub restart re-queues the job
- [ ] The worker container is on the internal model network only: it can reach the hub's gateway port but not the internet. It has a read-only root, tmpfs scratch, and only its own job folder mounted.
- [ ] A request that names a network or extra mount is rejected and logged.
- [ ] The image used is the digest from the launcher's config, even if the request names another image.
- [ ] The `local` backend runs the same dummy job, and the UI shows the isolation warning.
- [ ] With two users queueing jobs, neither waits behind the other's whole backlog (round-robin).
- [ ] The `docker` backend runs the dummy job against a Docker socket.

## Open questions

- SELinux is enforcing on the target host (Fedora 44). Per-job bind mounts need correct labels (`:Z` for a private job folder), or the launcher should use per-job named volumes instead. Pick one, and add a test with SELinux enforcing.

- The launcher is described as "a small container." Should it be the `searchy` image with a `launcher` role, or a separate tiny image? The same image keeps the "one image" promise.
- How does the launcher learn image digests on auto-update? Pinning by digest conflicts with `podman auto-update` on `:latest` unless the launcher resolves the digest at its own startup.
- The fair-scheduling gap item from PLAN.md is absorbed here. Confirm that.
