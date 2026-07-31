# Realty Ops

**A daily executive dashboard for real estate operators, built as a native desktop app.**

Real estate brokerages run on scattered tools — calendar in one place, email in
another, a CRM nobody fully trusts. Realty Ops pulls the pieces that matter
into one screen a broker actually opens every morning: what's on today,
what's coming up, and what needs attention first — generated automatically
from their real Google Calendar and Gmail, no manual entry.

This repo is a portfolio showcase — screenshots and a write-up only. The
source is closed while the product is pre-revenue.

## Latest release

**0.1.0-alpha.4** — 2026-07-31. [Full release notes](https://github.com/Wiinis/realty-ops/releases/tag/v0.1.0-alpha.4).

### Added

- Public staging deploy (`staging.wen-yen.xyz`): the renderer built standalone
  as a static site, no backend, no Electron shell. Reskinned as a fictional
  Honolulu portfolio ("Meridian Property Group") via a new `staging` customer
  config and `src/data/stagingMockData.js`, selected at build time by
  `CUSTOMER_ID` — real customers keep the original placeholder data
  (`src/data/defaultMockData.js`), `mockData.js` is now just the selector
  between the two. Non-functional screens (Connections, Refresh, the "Connect
  Google" banner) are hidden behind a `liveBackendEnabled` config flag rather
  than shipped as dead clicks. Deployed via Cloudflare Workers static assets
  (`wrangler.jsonc`, `cloudflare/staging-worker.js`), gated behind HTTP Basic
  Auth (`run_worker_first: true` is required for the auth check to actually
  run — by default Workers serves matching static assets straight from
  Cloudflare's edge cache without invoking the Worker at all). `npm run
  build:staging` / `npm run deploy:staging`.

### Fixed

- `DayLine.jsx` crashed the whole renderer on any calendar item with a null
  `time` (all-day events) — real Google Calendar data hits this immediately,
  mock data never did, so it only surfaced once a real customer connected.
  No error boundary existed to catch it, so the failure was a blank white
  window. Untimed items are now filtered out of the hourly timeline strip
  (they still show correctly everywhere else — Next 7/30 Days, etc.).
- Disconnecting Google only cleared the OAuth token, not the cached CEO
  Dashboard result, so the last real brief kept being served after
  disconnect. Backend now purges the cached brief on disconnect; the
  renderer also resets its live state immediately instead of waiting for a
  restart.
- "Refresh" and the daily scheduler only ever re-read the cached brief —
  nothing asked the backend to regenerate from Google, so reconnecting and
  refreshing kept coming back empty. Added an on-demand refresh endpoint
  (`POST /customers/:licenseKey/ceo-dashboard/refresh`) and wired the app to
  hit it live, falling back to the cache only if that call fails.

### Added (dev)

- `npm run server:demo`, a shortcut for `BACKEND_URL=http://localhost:8787
  npm run dev` — the app previously required remembering to set that env var
  by hand for local testing against a real backend.

## [0.1.0-alpha.3] - 2026-07-28

### Added

- Backend service (`backend/`): license activation/status, Stripe Checkout
  + billing webhooks, Google OAuth (Calendar + Gmail, read-only) with
  token storage and auto-refresh, and `scheduler.mjs` polling every 15
  minutes to generate each connected customer's CEO Dashboard once they
  cross their own local 7am. `dayLine`/`topFive`/`upcomingEvents` are now
  real once a customer activates a license and connects Google — see
  `backend/README.md` and `docs/business/ship-checklist.md`.
- `ceoDashboard.mjs`: generates the CEO Dashboard deterministically from
  Calendar/Gmail data — no LLM, no per-customer API cost. An earlier
  Anthropic Managed Agents version that judged importance across meetings
  and email is preserved on the `managed-agents-pipeline` git branch.
- `npm run start-realty-ops-server`: brings up the backend and a
  Cloudflare Tunnel together for exposing a local backend during a demo,
  waiting for the backend to be healthy before starting the tunnel.
- Connections screen: license activation and a "Connect Google" flow
  (`src/components/Connections.jsx`), talking to the backend via
  `electron/backendClient.js`.
- Legal drafts (`docs/legal/`) and a pricing proposal
  (`docs/business/pricing.md`) — both starting points, not finished
  decisions; see their own inline notes.

## Screenshots

### Today's Brief

Real data, not a mockup — pulled live from a connected Google Calendar. A
day timeline, a ranked Top Five, and the week and month ahead, all generated
without any manual input.

![Today's Brief](./screenshots/today-brief.png)

### Connections

Where the Google account gets linked, alongside an honest per-tile status
for the rest of the dashboard's data sources — some tiles ship live, others
are marked plainly as not yet connected rather than faked.

![Connections](./screenshots/connections.png)

## How it's built

- **Desktop app:** Electron + React (Vite), packaged as native installers
  for Windows and macOS via `electron-builder`. Sandboxed renderer
  (`contextIsolation`, no `nodeIntegration`) — the only bridge to the
  filesystem or network is an explicit preload script.
- **Daily brief generation:** deterministic, not LLM-based. A small backend
  service reads the connected Google Calendar and Gmail directly and shapes
  the brief with fixed rules — today's timed events, a ranked Top Five —
  rather than paying for a model call per customer per day. An
  Anthropic-Managed-Agents-based version that judged importance across
  meetings and email exists on a separate branch, held in reserve for once
  there's revenue to justify the added API cost.
- **Licensing & billing:** license activation, Stripe Checkout and billing
  webhooks, and a background subscription check that shuts the dashboard
  down gracefully (never abruptly) if a subscription lapses.
- **Multi-customer config:** one codebase, one `master` branch. Per-client
  branding, enabled panels, and brief delivery time are all just a config
  file — no per-customer forks.
- **Scheduler that survives real machines:** the daily-delivery timer
  re-arms on a short ceiling and re-reads the wall clock on every wake,
  specifically so it survives a closed laptop lid or a sleep cycle without
  double-firing or silently missing a day.
- **Auto-update & CI/CD:** every tagged release builds signed-pending
  Windows and macOS installers in CI and publishes them straight to GitHub
  Releases; the app checks for updates on startup and installs on next
  restart.
- **Tested:** a `node:test` suite covering the scheduler, config merging,
  connection persistence, and layout logic, plus a cross-platform smoke
  test that launches the packaged app and checks it survives — both gate
  every release.

## Status

Early alpha. The daily brief (calendar, email, Top Five) is real; several of
the dashboard's other panels are still placeholder data pending their own
integrations. Built solo, iterating toward a first paying customer.
