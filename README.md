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

**0.1.0-alpha.5** — 2026-08-14. [Full release notes](https://github.com/Wiinis/realty-ops/releases/tag/v0.1.0-alpha.5).

### Added

- Agent Workspace: a market-research and chat advisor backed by an Anthropic
  Managed Agent (pinned to v6 of the agent, SSE-streamed), reachable from the
  dashboard via `backend/routes/agent.mjs`.
- A general Agent Work Board (`npm run agent-board`) for visually tracking
  Codex CLI sessions and subagents across all local projects, supplemented by
  the user-level machine-readable `.codex/worker-status.json` feed.
- Five property-management panels: `rentDue`, `rentRoll`, `invoices`,
  `managerAgent`, and `documentIntake`. `rentDue` and `invoices` are no
  longer hand-typed placeholders — both are now derived deterministically
  (`config/lateFeeCalc.js`, `config/invoiceGen.js`) from each tenant's real
  `rentDueDay`/`lateFeeSchedule` and the rent roll (R1, R2/R7).
- A live Telegram intake/outbound channel (sibling `Agent API` repo,
  `src/telegramIntake.js`): text or voice messages to the bot are logged,
  transcribed (voice, via Workers AI — no separate OpenAI key), and answered
  with a real grounded Q&A reply (Claude Haiku over a static snapshot of
  this repo's rentRoll/rentDue/invoices).
- Manager Agent, project-manager sub-role (R6): an 8am cadence
  (`backend/managerAgentScheduler.mjs`) sends Joko a real Telegram check-in
  asking for a construction-project status update.
- Manager Agent, assistant sub-role (R5): sweeps unread Gmail, classifies
  and drafts replies via a new Agent API route (`POST /v1/email-draft-agent`,
  generic wording only — never states a balance/date it wasn't given), and
  creates real Gmail drafts (never sends). The 9am/3pm schedule itself ships
  disabled (`enabled: false` in `CADENCES`) pending a decision on when to
  turn it on.
- Print-at-home delivery for invoices (R2/R7): texting the bot a request
  like "send me a copy of an invoice" is classified (Haiku, matched only
  against real invoice ids), formatted as plain text, and queued on a new
  `print_jobs` table on this repo's backend. Each desktop app polls its own
  queue every 5 minutes (`electron/printJobsSyncDaemon.js`) and writes
  pending jobs into `userData/print-queue/`, where the existing local print
  daemon (`electron/printQueue.js`) sends them to the OS printer.
- Auto-updater activated: `electron/updater.js` now carries a real,
  read-only, repo-scoped GitHub token, so installed copies of the app can
  actually check GitHub Releases and self-update. This release itself still
  needs a manual install (the currently-installed copy has no token baked
  in) — every release after this one delivers automatically.
- Per-tile connections: `gmail`/`google_calendar` rows in the Connections
  screen now piggyback on the CEO Dashboard's single shared Google
  connection instead of showing "Needs API credentials", with a per-row
  Remove alongside the existing panel-wide Clear all. Added a "Log out"
  action, separate from disconnecting Google.

### Changed

- Connections screen: per-tile rows are now grouped into "Needs attention"
  and "Connected" sections (with counts) instead of one flat list, with a
  short intro sentence explaining what's actually wired up today. The full
  connector catalog browser is now a closed-by-default `<details>` instead
  of always rendering ~30 list items below the fold.
- Connections screen also gained review-only remediation for ambiguous
  legacy panel-connection data left over from the panel consolidation below
  — retained sources and folder paths are shown in full, with an explicit
  Replace-or-Remove per conflict, never auto-resolved.
- `pipeline`'s data shape gained `activeListings`, `pendingListings`,
  `newThisWeek`, and `priceDrops` fields, rendered as additional rows in the
  Pipeline tile, absorbing everything the removed `listings` panel had that
  Pipeline didn't already cover (see Removed).
- Rebuilt both mock datasets from scratch rather than tweaking values in
  place, and deliberately split their intent. `src/data/defaultMockData.js`
  (what every real, non-staging customer sees) now spans a deliberate spread
  of edge cases it never exercised before — empty arrays, zero counts, a
  null-time all-day calendar event, dollar-amount outliers, 60+ character
  strings — so panel empty-states, zero/"Clear" fallback branches, currency
  formatting at both extremes, and text wrapping all get exercised by mock
  data instead of only showing up the first time a real customer's data hits
  them. `src/data/stagingMockData.js` deliberately goes the other way and
  stays flattering: it's what prospects see live at staging.wen-yen.xyz, so
  it keeps non-empty showings/documents lists, a non-negative cash-position
  delta, fully-staffed manager-agent specialists, and a positive market
  trend — none of the new edge cases apply there.
- The Telegram Q&A route moved from Opus to Claude Haiku 4.5 — a grounded
  lookup over a small static snapshot never needed the larger model.

### Removed

- The `meetings` panel. Its three entries (08:30 standup, 12:00 lunch with
  lender, 19:00 recap call) were verbatim duplicates of the hardcoded
  DayLine timeline's meeting pins — same times, same titles, nothing extra.
- The `tasks` panel. Its title/dueBy pairs duplicated the hardcoded TopFive
  list, with every task's `owner` hardcoded to "You" and no field TopFive
  didn't already expose.
- The `listings` panel as a standalone tile. Its `underContract` count was a
  literal duplicate of Pipeline's "Under contract" stage count (both `5` in
  the current mock data); its remaining fields (`active`, `pending`,
  `newThisWeek`, `priceDrops`) moved into `pipeline` instead of being lost
  (see Changed).

Net panel count: 12 before this release cycle, +5 property-management
panels, -3 removed for redundancy = 14.

### Fixed

- The Advisor returned "invalid response" on every request — the backend's
  expected Managed Agent version pin had gone stale after the agent was
  bumped to v6 upstream. Both sides now agree on v6.

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
