# Realty Ops

**A daily executive dashboard for real estate operators, built as a native desktop app.**

Real estate brokerages run on scattered tools — calendar in one place, email in
another, a CRM nobody fully trusts. Realty Ops pulls the pieces that matter
into one screen a broker actually opens every morning: what's on today,
what's coming up, and what needs attention first — generated automatically
from their real Google Calendar and Gmail, no manual entry.

This repo is a portfolio showcase — screenshots and a write-up only. The
source is closed while the product is pre-revenue.

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
