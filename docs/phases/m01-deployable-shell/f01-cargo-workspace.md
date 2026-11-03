# F1 — Cargo workspace and role binary

**Milestone:** M1 Deployable shell · **Status:** ⬜ Not started · **Depends on:** — · **Unblocks:** [F2](f02-container-image.md), [F4](f04-accounts-auth.md), [F5](f05-projects-data-model.md), [H1](../mh-pipeline-core/h01-pipeline-core-types.md)

[← Back to plan](../PLAN.md)

## Goal

Set up the Rust workspace and the single `searchy` binary that every later feature builds on. The same binary runs as the hub or as a worker, depending on its first argument. This is the skeleton that makes "one image, several roles" possible.

## Scope

**In scope**

- One Cargo workspace for all server-side code.
- One `searchy` binary with subcommands `hub`, `fetcher`, `reader` and `analyst`.
- `searchy hub` starts an `axum` + `tokio` server and serves one HTML page.
- Worker subcommands that start, log, and exit cleanly (they are placeholders for now).
- A `/healthz` endpoint on the hub.
- Logs to stdout.
- An empty `searchy-pipeline` crate in the workspace. Its contents (stages, classifiers, counselors, the tracer and the chat loop) arrive in [H1](../mh-pipeline-core/h01-pipeline-core-types.md) and the rest of the H-series.

**Out of scope**

- Container image and CI → [F2](f02-container-image.md).
- Login and sessions → [F4](f04-accounts-auth.md).
- Database, migrations and real pages → [F5](f05-projects-data-model.md).
- Real worker behavior: fetcher → [F17](../m04-polite-web/f17-fetch-proxy.md), reader → [F11](../m02-local-knowledge/f11-notes-ingestion.md), analyst → [F26](../m06-async-research/f26-datasets-analyst.md).
- The `backup` role → [F30](../m07-idea-clouds-export/f30-export-backup.md).

## Design notes

- Server side is Rust wherever it makes sense, in one Cargo workspace. (Design: Overview; Tech stack)
- The image runs as `hub`, `fetcher`, `reader` or `analyst` depending on its first argument. A `backup` role is added later. (Design: Deployment › One image, several roles; Upgrades and operations)
- Web server: `axum` + `tokio`. Templates: `askama`, compiled in. Interactivity: htmx + SSE, with no frontend build step. (Design: Tech stack)
- `hub` exposes `/healthz` for container health checks. Logs go to stdout, so `journalctl` and `docker logs` just work. (Design: Deployment › Upgrades and operations)
- The binary must be static. F2 builds it with musl. (Design: Feature breakdown, F2)
- Harness core crates provide stages, classifiers, profiles, the tracer and the project chat loop. (Design: Tech stack) The chat harness was never built as a separate project, so these now live in the in-repo `searchy-pipeline` crate, built by the H-series and adapted from [`HarnessDesign.md`](../../appendix/HarnessDesign.md).
- Only the Fetcher touches the network. The Reader and Analyst work offline. (Design: Architecture)
- Crate maturity changes quickly. Check each crate when its feature starts. (Design: Tech stack)

## Implementation sketch

Suggestions only:

- Workspace layout:
  - `crates/searchy` — the binary with `main.rs` and `clap` subcommands.
  - `crates/searchy-hub` — the axum app, routes and templates.
  - `crates/searchy-worker` — shared worker scaffolding (job input/output, exit codes).
  - `crates/searchy-core` — shared types and config.
  - `crates/searchy-pipeline` — the stage pipeline. F1 creates it empty; [H1](../mh-pipeline-core/h01-pipeline-core-types.md) fills it.
- Use `tracing` + `tracing-subscriber` for stdout logs. Pick plain or JSON output with an env var.
- Hub config comes from env vars or a small TOML file: bind address (default `0.0.0.0:8080` inside the container) and data dir (default `/data`).
- Add a minimal askama base layout with the htmx script tag, vendored under `static/` and served by the hub. Making it phone-first from day one means one column and a viewport meta tag.
- Worker subcommands: parse args, log "started role X", exit 0. Add a `--version` flag.
- Add a `rust-toolchain.toml` and a CI check (`cargo fmt`, `clippy`, `test`). F2 adds image publishing on top.
- Add AGPL license headers or an SPDX field in `Cargo.toml` to match the AGPL decision.

## Acceptance criteria

- [ ] `searchy hub` serves a page; `searchy fetcher` runs and exits
  - (The design's "harness core crates imported" now means the in-repo `searchy-pipeline` crate from H1. Here it only has to exist and compile.)
- [ ] `searchy reader` and `searchy analyst` also run and exit 0.
- [ ] `GET /healthz` returns 200.
- [ ] An unknown role prints usage and exits non-zero.
- [ ] Logs go to stdout with a timestamp and level.
- [ ] `cargo build`, `cargo clippy -- -D warnings` and `cargo test` pass on a clean checkout.

## Open questions

- Decide whether workers share the hub's binary entry point or live in separate crates linked into one binary. Either works for the one-image model.
