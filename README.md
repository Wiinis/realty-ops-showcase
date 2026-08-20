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

**0.1.0-alpha.8** — 2026-08-20. [Full release notes](https://github.com/Wiinis/realty-ops/releases/tag/v0.1.0-alpha.8).

### Changed

- Operations & Documents folder scan is now manual, not automatic: the
  panel used to re-scan the connected folder on every mount (including
  every trip back from Connections), which is wrong for a large or slow/
  network folder. Scanning now runs only from an explicit "Scan folder"
  button next to "Review queue"; connection status is still checked
  automatically (cheap, reads persisted `connections.json` only). A
  connected-but-never-scanned-this-session state renders its own honest
  "Not scanned yet" prompt instead of mock rows or a misleading zero count.
  Rent & Collections gets the same manual-scan capability, without
  Operations' mock fallback since it never had placeholder rows.
- R5 Manager Agent assistant email cadence (Gmail sweep → classify → draft,
  never sends) is re-enabled — reverses the 2026-08-14 "keep disabled"
  decision by explicit request. Runs at 8am and 2pm in each customer's
  configured local timezone (was 9am/3pm, disabled).
- New customers now default to `Pacific/Honolulu` instead of
  `America/New_York`; existing rows are migrated so scheduled
  notifications don't fire six hours early for Hawaii-based customers.

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
