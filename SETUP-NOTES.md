# BookingPilot status page

Everything in this directory is copied into a **separate** repository
(`martingoga/status`, created from the `upptime/upptime` template). It is
kept here so the config, page code, and branding stay version-controlled
next to the app they describe.

## Architecture

- **Upptime** (GitHub Actions on the status repo) does the monitoring:
  pings endpoints every ~5 min, writes `history/summary.json` (incl.
  `dailyMinutesDown` + avg response times), response-time graphs under
  `graphs/`, and opens/closes GitHub issues for incidents.
- **The page itself** is `docs/index.html`: a hand-built BookingPilot-styled
  static page (no framework, inline CSS/JS) served by GitHub Pages from
  `master /docs`. It reads `summary.json` for component status, downtime
  and response times, and the issues API only for the incident cards.
- Upptime's own generated UI on `gh-pages` exists but is **not** served —
  Pages source is `master /docs`, so ours wins and the markup/layout is
  100% ours. The Upptime UI would only resurface if the Pages source is
  switched back.
- Runs entirely on GitHub + the user's browser, so it stays up even when
  BookingPilot/Supabase is down.

## Files

- `.upptimerc.yml` — monitor definitions and Upptime's own (unused) site
  config
- `docs/index.html` — the status page (this is what people see)
- `docs/maintenance.json` — optional; `{"active":true,"message":"...",
  "until":"<ISO>"}` shows a maintenance banner, auto-expires at `until`
- `docs/fonts/` — self-hosted Host Grotesk woff2
- `docs/logo.svg`, `docs/icons/` — brand assets used by the page
- `assets/` — same assets in Upptime's expected location (logo/theme are
  referenced by `.upptimerc.yml` URLs; the `?v=` cache-bust on `themeUrl`
  only matters if we ever switch back to the Upptime UI)

## Editing the page

Edit `docs/index.html` (and `docs/` assets), push to `martingoga/status`
master, wait ~1 min for Pages to redeploy. No workflows involved —
Pages serves the files as-is.

## Monitors

Defined in `.upptimerc.yml` under `sites:`. All `check=` URLs hit the
public `health` edge function (`verify_jwt = false`, returns only an
ok/error signal):

- Web app — the SPA URL (kept red until `bookingpilot.al` ships — see below)
- API & database — `?check=` (default): real Postgres round-trip
- Authentication — `?check=auth`: probes GoTrue server-side
- Reservations — `?check=reservations`: booking pipeline read
- Realtime — `?check=realtime`: websocket endpoint liveness
- File storage — `?check=storage`: `hk-attachments` bucket listing
- Email delivery — `?check=email`: Resend reachable + `RESEND_API_KEY` set
- Channel manager — `?check=channex`: api.channex.io + key **validity**
  (a configured key must get a 200 — a revoked key now reads as down)
- Schedulers — `?check=jobs`: pg_cron dead-man's switch, reads
  `public.monitor_heartbeats` (bumped every 5 min by cron); stale >20 min
  = scheduler/DB-write path dead
- Sync pipeline — `?check=sync`: `channex_connections.last_sync_at`
  freshness for connected properties; stale >20 min = OTA bookings stalled
- Webhooks & hooks — `?check=edge`: send-email, channex-webhook,
  resend-webhook, guest-checkin, unsubscribe all deployed & answering
  (4xx/405 = up; 404 = undeployed; 5xx = broken)
- AI assistant — `?check=groq`: api.groq.com + `GROQ_API_KEY` validity
- Status page — `martingoga.github.io/status` itself (catches Pages
  outages; cannot catch a GitHub Actions outage — nothing would run)

Two **direct** probes bypass the edge function entirely, so a page full of
red with these still green means "edge runtime down", not "platform down":

- Gateway & PostgREST — `rest/v1/rpc/health_ping?apikey=<anon>` → 200 from
  a data-less ping function (proves Kong→PostgREST→DB without granting anon
  any table access)
- Auth service (direct) — `auth/v1/health?apikey=<anon>` → expects 200

The anon key in those URLs is the public client key (in every browser
bundle) — safe to keep in this public config.

Deploy the health function before enabling monitors:
`supabase functions deploy health`

## Alerting

None configured — monitors open/close GitHub issues on `martingoga/status`
and the status page shows the state, but nothing pages you. Watch the repo
or the page. To add a channel later, Upptime supports email (SMTP/Resend)
and Slack/Discord webhooks via `NOTIFICATION_*` repo secrets plus a
`secrets:` allowlist in `.upptimerc.yml`.

## Required secret

`GH_PAT` (repo secret, Actions): a classic PAT with `repo` + `workflow`
scopes — Upptime's workflows need it to commit status data. If it expires,
checks stop silently — the page shows a "Monitoring delayed" banner once
`summary.json` is >20 min stale, but set a reminder to rotate it anyway.

## When bookingpilot.al is registered

- DNS: `status.bookingpilot.al` → CNAME → `martingoga.github.io`
- The page works unchanged on the custom domain (all asset paths are
  relative; only the GitHub API/raw URLs are absolute).
- `.upptimerc.yml`: set `cname: status.bookingpilot.al`, drop `baseUrl`,
  switch logo/favicon/icon/theme URLs to `https://status.bookingpilot.al/…`.
- **Reset the Web-app history** so pre-launch red doesn't poison uptime:
  delete `history/web-app.yml` + `graphs/web-app/` on the status repo and
  close its incident issue.
- Repoint the "Report an issue" links in `docs/index.html` back to
  `mailto:support@bookingpilot.al` once the mailbox exists.

## Caveats

- Scheduled Actions are best-effort: 5-min checks can slip under load —
  the stale-data banner covers this visibly.
- `raw.githubusercontent.com` caches ~5 min, so the page reflects the
  last completed check cycle.
- Incident timeline = GitHub issues on the status repo (label `status`);
  write incident updates as issue comments.
- GitHub's issues API is unauthenticated (60 req/h per IP) — only the
  incident cards use it; uptime bars/% come from `summary.json` and are
  unaffected by rate limits.
