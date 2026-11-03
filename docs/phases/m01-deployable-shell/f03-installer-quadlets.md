# F3 — Installer, Quadlets and compose file

**Milestone:** M1 Deployable shell · **Status:** ⬜ Not started · **Depends on:** [F2](f02-container-image.md) · **Unblocks:** —

[← Back to plan](../PLAN.md)

## Goal

Make installation one command on a Linux host. A short script creates an isolated `searchy` user, writes Podman Quadlet units (or a compose file), starts the hub, and prints the URL. Everything after that happens in the browser.

## Scope

**In scope**

- An installer shell script that:
  - detects Podman or Docker;
  - creates the dedicated `searchy` user, or reuses it if it already exists, and enables lingering;
  - enables the user's `podman.socket` for the launcher;
  - checks that the hub port is free, and lets you choose another;
  - writes the units or compose file;
  - starts the services and prints the URL.
- Quadlet `.container`, `.volume` and `.network` units for the hub, the launcher sidecar and the networks: an internal model network with no internet route, which workers use to reach the hub's model gateway (D4), and the Fetcher's proxy network.
- A compose file with the same shape, for Docker users and quick trials.
- Volumes: `searchy-data`, `searchy-corpora` and `searchy-launcher` (the shared socket dir). There is no notes mount; notes are uploaded in the browser (D6).
- A listen option: `127.0.0.1` by default, or the LAN with `--lan` so phones on the local network can reach the hub over plain HTTP (D3).
- Docs for the optional HTTPS path with `tailscale serve`, which turns on passkeys, Web Push and PWA install (D3).
- Auto-update labels.
- SELinux-correct volume and mount labels; Fedora runs SELinux in enforcing mode.
- A way to reach model servers and SearXNG running on the host itself, via `host.containers.internal`.
- An uninstall path (suggested).

**Out of scope**

- The launcher sidecar's code → [F6](f06-job-queue-launcher.md). F3 only ships its unit, wired up once F6 exists.
- The first-run browser setup → [F4](f04-accounts-auth.md) (admin account) and [F8](../m02-local-knowledge/f08-model-routing.md) (models).
- Kiwix and SearXNG units → [F10](../m02-local-knowledge/f10-kiwix-corpus-manager.md), [F19](../m04-polite-web/f19-searxng.md).
- Cloudflare Tunnel or reverse-proxy docs. This is a gap item in PLAN.md, suggested as a docs task alongside F3.

## Design notes

**Install paths** (Design: Deployment › Two supported install paths)

| Path | Best for | How |
| --- | --- | --- |
| Podman Quadlets (recommended) | Fedora, Bazzite, any systemd host | Write `.container`, `.volume`, `.network` to `~/.config/containers/systemd/`, then start them |
| Compose file | Docker users, quick trials | `docker compose up -d` or `podman compose up -d` with the same file |

**Minimal hub Quadlet** (Design: Deployment)

```ini
[Container]
Image=ghcr.io/<owner>/searchy:latest
Exec=hub
Volume=searchy-data:/data
Volume=searchy-corpora:/corpora
Volume=searchy-launcher:/run/searchy-launcher
PublishPort=127.0.0.1:8080:8080
Label=io.containers.autoupdate=registry

[Service]
Restart=always

[Install]
WantedBy=default.target
```

**Isolation** (Design: Deployment › Isolation, in two layers)

- The installer creates a `searchy` Linux user. This is the one step that needs sudo. It enables lingering so services run without a login, and installs the Quadlets in that user's systemd folder. A compromise reaches only that account, never your home folder.
- The hub never holds the container socket. Only the launcher sidecar mounts the rootless Podman socket.

**Storage** (Design: Deployment › Storage)

- `searchy-data`: SQLite, indexes and the WARC archive. This is the volume to back up.
- `searchy-corpora`: big dumps. It can live on a slow disk and is skipped in backups.
- Uploaded documents live in `searchy-data`. There are no host folder mounts (D6).

**Access** (Design: Web interface › Remote access and auth): reach the hub over Tailscale or the local network, with nothing exposed to the internet. The default unit publishes on `127.0.0.1`. The `--lan` option publishes on the LAN instead.

**HTTPS is optional** (D3): over plain HTTP the hub uses password login and ntfy. Browsers allow passkeys, Web Push and PWA install only in a secure context, so those turn on when the hub is served over HTTPS, for example with `tailscale serve`. `http://<lan-ip>` is not a secure context.

**Networks** (D4; Design: Deployment › Isolation): workers join only an internal Podman network (`podman network create --internal`), where the hub's model gateway is the only reachable service. The hub joins that network and the default one. The Fetcher also joins the proxy network.

**Upgrades** (Design: Deployment › Upgrades and operations): `podman auto-update` or Watchtower. Logs go to stdout and `journalctl`.

## Implementation sketch

Suggestions only:

- `install.sh`, in POSIX sh. Steps:
  1. Check the OS, systemd, and Podman ≥ the Quadlet-capable version (or Docker).
  2. If `searchy` doesn't exist, `sudo useradd --system --create-home searchy`. Either way, check that subuid/subgid entries exist for rootless Podman.
  3. `sudo loginctl enable-linger searchy`, then `systemctl --user enable --now podman.socket` as that user. The socket is off by default on Fedora.
  4. Render the units into `~searchy/.config/containers/systemd/`.
  5. Run `systemctl --user daemon-reload` and `start searchy-hub` as that user (via `machinectl shell` or `sudo -u searchy XDG_RUNTIME_DIR=...`).
  6. Wait for `/healthz` and print the URL.
- `--port N` sets the host port; 8080 is only the default. Check it with `ss -tln` before writing units, since 8080 is commonly taken (by Jetty, Tomcat and others).
- `--lan` changes `PublishPort` to `0.0.0.0:<port>:8080` (or a given address). Print both the LAN URL and a note that passkeys and push need HTTPS.
- Write `searchy-models.network` with `Internal=true` and `searchy-proxy.network`. The hub unit joins `searchy-models`.
- Enable `podman-auto-update.timer` for the searchy user.
- Ship templates in `deploy/quadlet/` and `deploy/compose.yaml`. Keep them in one source of truth if practical.
- Test in a Fedora VM, with a script or CI job using a cloud image.
- Add `install.sh --uninstall`: stop the units and remove them. Keep volumes unless `--purge` is passed.

## Acceptance criteria

- [ ] Fresh Fedora VM: one command to a working login page
- [ ] Services run as the `searchy` user and survive a reboot with nobody logged in (lingering works).
- [ ] The hub container has no container socket mounted.
- [ ] By default the hub port is bound to `127.0.0.1` only. With `--lan`, a phone on the local network reaches the login page over plain HTTP and can sign in with a password.
- [ ] The `searchy-models` network is internal: a test container on it can reach the hub but not the internet.
- [ ] `docker compose up -d` with the shipped compose file also reaches the login page.
- [ ] Re-running the installer is safe (idempotent), including on a host where the `searchy` user already exists.
- [ ] If the default port is in use, the installer says so and offers another, rather than failing at container start.
- [ ] With SELinux enforcing, the hub can read and write its volumes and reach a model server on the host through `host.containers.internal`.

## Open questions

- The "working login page" depends on [F4](f04-accounts-auth.md). F3 can be built before F4 by verifying a placeholder page, then re-checked once F4 lands.
- Resolved (D6): there is no notes folder to share with the `searchy` user. Notes are uploaded in the browser.
- Resolved (D3): HTTPS is optional. Document `tailscale serve` as the easy path.
- In compose mode the dedicated-user isolation is weaker. Document the trade-off.
- Cloudflare Tunnel or reverse-proxy docs: gap item in PLAN.md.
