# PRD-0001: Freebuff Token Auto-Refresh

## Problem Statement

The freebuff proxy at `/home/hermes/freebuff-proxy/proxy.js` rotates auth tokens against `codebuff.com`. Each token is locked to a `(email, IP, token)` tuple — every fresh login from a previously-used `(email, IP)` returns `409 model_locked` and the session gets pinned to a single model (typically `deepseek/deepseek-v4-flash`). The only known escape is to authenticate with a **new email** and through a **fresh IP** (VPN).

Today this re-auth workflow is fully manual: Tim opens a browser, signs into freebuff.com with a different email, copies a callback URL back into the proxy dashboard, hopes nothing broke, and finally restarts the systemd service. When the 24h credential expiry fires while he's asleep (e.g. tonight at 12:50 AM PT), the 8 AM morning-briefing cron fails silently and the entire Telegram chat with Hermes goes dark until manual intervention.

## Solution

A new service, `freebuff-auth-automation`, watches the proxy token's health, and when it goes stale, runs an unattended headless re-auth flow:

1. Calls `POST /api/auth/start` to mint an `auth_code`
2. Rotates the system VPN to a new exit IP (so the `(email, IP)` tuple is fresh)
3. Opens `https://freebuff.com/login?auth_code=…` in a headless Chromium with a stored credential
4. Submits email + password, captures the `freebuff://` (or http callback) URL from the post-login redirect
5. Calls `POST /api/auth/status` with `{fingerprintId, fingerprintHash, expiresAt}` until `data.user.authToken` appears
6. Appends the new `authToken` to `/home/hermes/freebuff-proxy/.config/config.json`
7. Restarts `freebuff-proxy.service` via `systemctl`
8. Logs the new token's email + expiry to a SQLite ledger

If anything in the chain fails, the watchdog alerts Tim on Telegram with the failing step and a one-line remediation. If multiple identities are saved, the service picks the next one alphabetically to keep load spread.

## User Stories

1. As the sole operator, I want the proxy token to refresh itself, so that my morning briefing always runs and I never have to log in by hand.
2. As the sole operator, I want the system to keep multiple freebuff identities in rotation, so that I am not blocked when one identity gets server-locked.
3. As the sole operator, I want re-auth to use a fresh IP per cycle, so that `model_locked` triggers from prior sessions cannot block the new login.
4. As the sole operator, I want credentials encrypted at rest with a key only my user can read, so that a leaked disk image never leaks my freebuff accounts.
5. As the sole operator, I want a `systemctl status freebuff-auth-automation` view, so that I can see at-a-glance how many tokens are valid, when the last refresh ran, and which identity is currently in use.
6. As the sole operator, I want the watchdog to alert me on Telegram when manual intervention is required, so that I learn about problems within minutes instead of at 8 AM.
7. As the sole operator, I want a CLI command `freebuff-auth refresh --now` so that I can force a refresh on demand (e.g. before a known outage window).
8. As the sole operator, I want the system to refuse to start if any required secret (VPN, email password, encryption key) is missing, so that failures fail loud.
9. As the sole operator, I want the headless browser flow to avoid using my real Chrome profile (no cookies, no history), so that the automation cannot accidentally leak other freebuff sessions.
10. As the sole operator, I want the proxy config to dedupe tokens on write, so that adding the same token twice does not double the load.
11. As the sole operator, I want each token's age and last-success timestamp recorded, so that I can predict when the next refresh will fire.
12. As the sole operator, I want the service to skip re-auth when the current proxy token is still healthy, so that we do not burn identities unnecessarily.
13. As the sole operator, I want a `dry-run` mode that prints the plan without executing any system changes, so that I can review an upcoming refresh before it lands.
14. As the sole operator, I want a `scripts/seed_identity.py` helper to add a new `(email, password)` to the encrypted vault, so that I can grow the pool on demand.
15. As the sole operator, I want the system to handle the case where freebuff.com changes the device-flow endpoints without warning, so that small upstream changes don't silently break me.

## Implementation Decisions

- **Project home**: `~/freebuff-auth-automation/` (already scaffolded). Repo: `NotRllyRn/freebuff-auth-automation`.
- **Runtime**: Node.js 20 (matches `freebuff-proxy/proxy.js`). Single-binary or `npm` script — no Python. Reuse `SocksProxyAgent`/`node-fetch` patterns where possible.
- **Headless flow**: Playwright (Chromium) launched with `--user-data-dir` pointing to a per-identity temp dir, so each run starts with no cookies.
- **Credential vault**: AES-256-GCM with the symmetric key read from `~/.config/freebuff-auth-automation/key` (`0600`, owned by `hermes`). Vault stored at `~/.config/freebuff-auth-automation/vault.json` (`0600`).
- **VPN control**: Thin abstraction over the active VPN client (TBD — Mullvad, Proton, or local wireguard with multiple peers). Module exposes `rotateTo(country?)` and `currentExitIp()`.
- **Proxy integration**: Detect proxy health by calling `https://www.codebuff.com/api/v1/freebuff/streak` with the current token. `error` field present → unhealthy. Run weekly to spread load.
- **Token write**: Edit `/home/hermes/freebuff-proxy/.config/config.json` in place via `lodash.set` semantics + `js-beautify` round-trip; preserve formatting. After write, run `systemctl restart freebuff-proxy.service`.
- **Systemd service**: `freebuff-auth-automation.service` (user-mode, `~/.config/systemd/user/`) so it runs without root. `Wants=network-online.target`. `Restart=on-failure`.
- **Watcher loop**: Built-in interval (NOT an external cron). Default 30-minute probe interval. Configurable via `~/.config/freebuff-auth-automation/config.json`.
- **Alerting**: Reuse existing `send_message` / curl-telegram-bot path. If unavailable, write to a dropbox file the daily-briefing cron picks up.
- **Logging**: JSON lines to `~/.local/share/freebuff-auth-automation/audit.log` with `pino`-style fields. Rotated by `logrotate`.
- **Persistence**: SQLite at `~/.local/share/freebuff-auth-automation/ledger.sqlite` with one row per `(email, token_added_at, last_seen_at, status, last_error)`.
- **CLI**: `bin/freebuff-auth` exposing at least `status`, `refresh`, `add-identity`, `list-identities`, `dry-run`.

## Testing Decisions

- **External-behavior only.** Tests poke the public CLI surface and the proxy health endpoint — no assertions on internal Map keys or class shapes.
- **Modules covered**: `VpnRotator`, `CredentialVault`, `TokenWriter`, `HeadlessAuthFlow`, `Watchdog`.
- **Test framework**: `vitest` (matches Node TS ergonomics, fast watcher, no transpile step).
- **Fixtures**:
  - Mock freebuff endpoints with `nock` or `msw` for the device-flow happy path AND the locked path (`409 model_locked`)
  - Record one real freebuff end-to-end playback (manual capture, stored under `fixtures/freebuff-real-flow.json`) for the integration test
- **Live "canary" probe**: An opt-in `--canary` flag that runs in dry-run against the real freebuff URLs but stops before the browser-form-submit step, to detect upstream drift without burning identities.
- **Test pyramid**:
  - Unit per module
  - One integration test for `HeadlessAuthFlow` against the recorded fixture
  - One end-to-end smoke that runs against a local mock proxy

## Out of Scope

- Multi-operator support (the system is a single-user tool).
- Browser-visible UI / dashboard — the existing `dashboard.html` already serves that surface; we only need a small status sub-page.
- Mobile push (Telegram-only alerts).
- Mirroring the same system to `coding-container` — that's a separate box; later we can package this as a Docker image if cross-host is wanted.
- Token rotation *inside* a single session (proxy already handles model switching within a session; we only handle cross-session refresh).

## Further Notes

- The proxy already exposes `POST /api/auth/start` and `POST /api/auth/status` (proxy.js lines 2343–2364). The callback URL is what we need to scrape from the browser post-login.
- The proxy supports a token pool (`config.authTokens` is an array, used round-robin). Adding a token without removing stale ones is safe — stale tokens 401 but do not crash the proxy.
- Today's `daily_briefing.py` is being switched to a script-driven cron (`no_agent=true`) so it remains unaffected by proxy outages. This PRD does not touch that schedule.
- See `docs/adr/` for design-record entries created during implementation.
