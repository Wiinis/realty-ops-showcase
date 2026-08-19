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

**0.1.0-alpha.7** — 2026-08-19. [Full release notes](https://github.com/Wiinis/realty-ops/releases/tag/v0.1.0-alpha.7).

### Added

- AI usage tracking (`GET /customers/:licenseKey/agent/usage`, 30-day
  rolling window, guarded by the existing `requireActiveCustomer` — no new
  auth layer): a new `ai_usage` table records one row per AI call, tied to
  the license key since Realty has no user/role model — a tenant-wide
  total plus a per-feature breakdown (`next_best_actions`, `chat`,
  `assistant_email_draft`), deliberately with no per-user rows or
  owner/non-owner scoping. Recording happens at a single choke point
  inside `runPortfolioAgent` (`backend/agentGateway.mjs`) so no call site
  can silently miss being counted. Today every bucket's token counts and
  `estimated_cost` come back `null` for every call — the managed-run
  protocol this backend speaks emits no usage field on any event — so in
  practice this ships as a call-volume/per-feature view, not a cost
  tracker, until that upstream gap closes; the parser already picks up
  real counts automatically if it does. See `backend/README.md`'s "AI
  usage" section.
- Operations & Documents folder scan: the desktop app can now scan a
  connected folder for the Operations panel (`electron/folderScan.js`) and
  the panel shows real files awaiting filing instead of demo rows once a
  folder is connected. Non-recursive, top-level only, allowlisted to
  `.pdf/.jpg/.jpeg/.png/.doc/.docx`, capped at 500 items per scan (with the
  true total still reported). No document classification exists yet, so
  scanned items show their real filename and an explicit "Not yet
  classified" state rather than a guessed type or destination. A folder
  that goes missing or becomes unreadable is reflected in Connections as
  "Folder not found" / "Folder unreadable" instead of silently reverting to
  "Not connected".
- Document classification backend + main-process wiring
  (`backend/documentClassification.mjs`, `electron/documentClassify.js`,
  new `POST /customers/:licenseKey/documents/classify` route, new
  `classify-panel-document` IPC channel): given a panelId and a bare file
  name from a prior folder scan, the connected folder's document is read in
  the main process, base64-encoded, and sent to the existing Agent API
  gateway (one document per call, its own `document_classify` usage
  feature) for a best-effort type/proposed-name/destination guess. Every
  filesystem read is guarded against symlinks with both `lstat` (never
  follows a link) and a `realpath` containment check against the connected
  folder, closing an arbitrary-file-read/exfiltration hole a symlink
  planted in the folder would otherwise open; `isEligibleName` also now
  rejects `:` to block NTFS alternate-data-stream names. Renderer wiring
  for this capability is not part of this release — the folder-scan panel
  still shows the explicit "Not yet classified" state end to end.

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
