# F4 — Accounts and auth

**Milestone:** M1 Deployable shell · **Status:** ⬜ Not started · **Depends on:** [F1](f01-cargo-workspace.md) · **Unblocks:** —

[← Back to plan](../PLAN.md)

## Goal

Add login to the hub. The first user becomes the admin, and others join by invite link. Everyone has a password, and can add a passkey when the hub is served over HTTPS (D3). Login is required even on a private network, because the hub can start containers. Sessions on a phone last for days, so research can be checked without signing in again.

## Scope

**In scope**

- First-run admin creation with a password, plus a passkey when served over HTTPS.
- Invite links created by the admin.
- Password login, hashed with argon2. This always works, including over plain HTTP.
- Passkey (WebAuthn) registration and login, offered only in a secure context (HTTPS or `localhost`). Over plain HTTP the passkey options are hidden (D3).
- Long-lived sessions on trusted devices, with the ability to sign out.
- Instance roles: admin and member.
- Auth middleware that protects every route except login, registration, `/healthz` and static assets.

**Out of scope**

- Project roles (owner, editor, viewer) → [F5](f05-projects-data-model.md).
- Settings for API keys and models → [F8](../m02-local-knowledge/f08-model-routing.md).
- Budgets per user → [F25](../m06-async-research/f25-fan-out-budget.md).
- The rest of the first-run wizard (models, corpora, notes). See the PLAN.md gap item.

## Design notes

**Auth** (Design: Web interface › Remote access and auth)

- Built-in login is required even on a private network: passwords always, plus passkeys (WebAuthn) when served over HTTPS.
- HTTPS is optional (D3). Browsers allow WebAuthn only in a secure context, and `http://<lan-ip>` isn't one.
- Sessions are long-lived on trusted devices, so the phone stays signed in.

**Accounts** (Design: Teams and permissions › Accounts)

- The admin sends invite links. Each person sets a password and, when the hub is served over HTTPS, registers a passkey.
- SQLite in WAL mode is enough for a small team.

**Instance roles** (Design: Teams and permissions › Roles)

| Role | Can |
| --- | --- |
| Admin | Invite users, set API keys and model endpoints, set budgets, manage corpora and profiles |
| Member | Create projects, upload personal notes and datasets |

**First run** (Design: Deployment › First run): step 1 is "Create the admin account with a password, and a passkey if the hub is served over HTTPS."

**Stack** (Design: Tech stack): `webauthn-rs`, `argon2`, SQLite via `sqlx`, `axum`, `askama`, htmx.

Remote access may be over Tailscale, or optionally a Cloudflare Tunnel or reverse proxy for a public URL. So the WebAuthn origin and RP ID must be configurable. (Design: Remote access and auth)

## Implementation sketch

Suggestions only:

- **Tables** (via `sqlx` migrations; F5 owns the migration setup, so coordinate if F4 lands first):
  - `users(id, display_name, instance_role, created_at, disabled_at)`
  - `passkeys(id, user_id, credential_json, name, created_at, last_used_at)`
  - `passwords(user_id, argon2_hash)`
  - `invites(token_hash, created_by, role, expires_at, used_by, used_at)`
  - `sessions(id_hash, user_id, device_label, created_at, last_seen_at, expires_at)`
- **First run**: if there are no users, `/setup` lets the first visitor create the admin. This is a race on an exposed host. Mitigate with a one-time setup token printed to the logs or by the installer.
- **Sessions**: an opaque random token in an `HttpOnly; SameSite=Lax` cookie, stored hashed. Add `Secure` only when the request arrived over HTTPS, or the cookie won't be stored over plain HTTP. Use a sliding expiry, e.g. 90 days and refreshed on use.
- **CSRF**: rely on SameSite plus an htmx header check (`HX-Request`) or a token on state-changing forms.
- **Config**: `SEARCHY_PUBLIC_URL` sets the WebAuthn RP ID and origin. Passkeys need HTTPS or `localhost`. Tailscale HTTPS certs (`tailscale serve`) are the easy path; document this.
- **Secure-context detection**: the page checks `window.isSecureContext` (and `PublicKeyCredential` support) before showing passkey buttons. The server also refuses WebAuthn ceremonies when the origin isn't HTTPS or `localhost`.
- **Pages**: login (password form; a passkey button in a secure context), register-from-invite, account (manage passkeys and sessions), admin users list (invite, disable, change role).
- Rate-limit password attempts per account and IP.

## Acceptance criteria

- [ ] Phone stays signed in across days
- [ ] The first visitor can create the admin only with the setup token, or only when no users exist.
- [ ] An invite link lets a new member set a password, and register a passkey when served over HTTPS. The link can't be reused or used after it expires.
- [ ] Password login works over plain HTTP on a LAN address, and the stored password is an argon2 hash.
- [ ] Over plain HTTP no passkey option is shown. Over HTTPS, passkey registration and login work.
- [ ] Every page except login, register, setup, `/healthz` and static assets redirects an anonymous visitor to login.
- [ ] A member can't reach admin pages (403).
- [ ] A user can see and revoke their sessions and passkeys.

## Open questions

- Resolved (D3): HTTPS is optional. Plain HTTP is password-only; passkeys appear over HTTPS.
- Account recovery when a user forgets their password: is an admin reset enough?
- First-run wizard steps beyond admin creation: see the PLAN.md gap item "First-run setup wizard."
