# F2 — Container image and multi-arch CI

**Milestone:** M1 Deployable shell · **Status:** ⬜ Not started · **Depends on:** [F1](f01-cargo-workspace.md) · **Unblocks:** [F3](f03-installer-quadlets.md)

[← Back to plan](../PLAN.md)

## Goal

Package the `searchy` binary as one small container image for amd64 and arm64, and publish it from CI. Users then run Searchy with no Rust toolchain, on anything from a desktop to a Raspberry Pi 5.

## Scope

**In scope**

- A static musl build of the `searchy` binary.
- A minimal container image with that binary as the entrypoint. The role is passed as the first argument.
- Multi-arch builds (linux/amd64, linux/arm64) and a published manifest list.
- A CI workflow that builds, tests and pushes the image to a container registry.
- An image healthcheck that uses `/healthz` for the hub role.

**Out of scope**

- Quadlet units, the compose file and the installer → [F3](f03-installer-quadlets.md).
- Pinning worker images by digest in the launcher → [F6](f06-job-queue-launcher.md).
- The Kiwix image, which is upstream → [F10](../m02-local-knowledge/f10-kiwix-corpus-manager.md).
- A reader image with Poppler for PDFs. Decide this in [F11](../m02-local-knowledge/f11-notes-ingestion.md).

## Design notes

- The whole system ships as one container image containing one static Rust binary. (Design: Deployment)
- The same image runs as `hub`, `fetcher`, `reader` or `analyst` depending on its first argument. Kiwix runs as its own upstream image. (Design: Deployment › One image, several roles)
- Images are published to a container registry for amd64 and arm64, so a Raspberry Pi 5 or a small ARM server works. (Design: Deployment › One image, several roles)
- The example Quadlet uses `Image=ghcr.io/<owner>/searchy:latest` and `Label=io.containers.autoupdate=registry`. So `latest` tags must be kept current for `podman auto-update`. (Design: Deployment)
- Upgrades: `podman auto-update` (or Watchtower for Docker) pulls new images. Database migrations run at hub startup. (Design: Deployment › Upgrades and operations)
- Workers run in no-network, read-only containers with tmpfs scratch, so the image must work with a read-only root filesystem. (Design: Deployment › Isolation)
- PDF text: `pdf-extract`, or Poppler in the reader image. Poppler is more robust. (Design: Tech stack)

## Implementation sketch

Suggestions only:

- Build with `cargo zigbuild` or `cross` targeting `x86_64-unknown-linux-musl` and `aarch64-unknown-linux-musl`. Native-runner builds per arch are another option. Avoid slow QEMU emulation for Rust compiles if possible.
- Watch for C dependencies that complicate musl, such as SQLite, DuckDB, `sqlite-vec` and TLS. Prefer `rustls` over OpenSSL, and bundle SQLite.
- Runtime base: `scratch` or `gcr.io/distroless/static`. Include CA certificates and timezone data, and run as a non-root UID.
- Add a `HEALTHCHECK` (or the Quadlet `HealthCmd`) that calls `searchy healthcheck`. That subcommand hits `/healthz` locally, since there's no curl in a scratch image.
- CI (e.g. GitHub Actions): on a push to `main` → `:edge`. On a tag `vX.Y.Z` → `:X.Y.Z` and `:latest`. Use `docker buildx` or `podman manifest` for the multi-arch manifest.
- Smoke-test each arch in CI: `podman run --rm --arch arm64 <img> fetcher` exits 0.
- Record image digests in the release notes. The launcher (F6) pins workers by digest.

## Acceptance criteria

- [ ] Image runs on amd64 and arm64
- [ ] The binary inside is statically linked (`file` or `ldd` confirms it).
- [ ] `podman run <img> hub` serves the page and passes its healthcheck.
- [ ] The image runs with `--read-only` and a tmpfs `/tmp`.
- [ ] The image runs as a non-root user.
- [ ] A tagged release publishes a multi-arch manifest to the registry without manual steps.

## Open questions

- **Name check** (PLAN.md open question 2): settle the name before publishing under a registry path.
- Decide whether to use one image for all roles, or a reader variant that includes Poppler. The design prefers one image. Poppler isn't static-friendly, so this may force a choice in F11.
- Registry choice: the design's example uses ghcr.io. Confirm the owner and namespace.
