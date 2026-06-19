# AGENTS.md — freebuff-auth-automation

Headless OAuth-device-flow re-auth for freebuff proxy tokens. Mints fresh tokens via freebuff.com/login, rotates `(email, IP, token)` tuples to escape server-side locks.

## Agent skills

### Issue tracker

Issues and PRDs live as GitHub issues on `NotRllyRn/freebuff-auth-automation`. **PRs are NOT a triage surface** (solo project). See `docs/agents/issue-tracker.md`.

### Triage labels

Default 5-string vocabulary: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context layout — one `CONTEXT.md` + `docs/adr/` at repo root. See `docs/agents/domain.md`.
