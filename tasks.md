# Tasks: Job Application Tracker

**Companion to:** `job-application-tracker-prd.md` (v1.1)
**How to use this file:** Each task is scoped to one Claude Code pairing session (roughly 1–3 hours). Work top to bottom within a phase; don't start a phase until the previous phase's exit criteria are met. Check boxes as you go, and add notes/decisions inline — this is a living document.

**Session ritual:** Start each session by telling Claude Code which task you're on and pointing it at the PRD. End each session with: code committed, the task's "done when" verified by hand, and one sentence noting anything you learned or changed.

---

## Phase 1 — Core Tracker

**Goal:** Fully replace your spreadsheet.
**Exit criteria:** You can add, view, edit, search, and change the status of applications, and every status change appears in the activity timeline.

### 1.1 Project skeleton & tooling

- [ ] Monorepo layout: `client/` (React + Vite + TS), `server/` (Node + TS), `shared/` (types)
- [ ] Express or Hono server with a `/health` endpoint; Vite dev proxy pointed at it
- [ ] ESLint + Prettier, `tsconfig` per package with a shared base
- [ ] npm scripts: `dev` (runs client + server concurrently), `build`, `lint`
- **Done when:** `npm run dev` starts both apps and the client can fetch `/health`.
- **Learning focus:** monorepo wiring, TS project references.

### 1.2 Database schema & migrations

- [ ] Install Drizzle + SQLite driver; configure Drizzle Kit
- [ ] Schema: `applications`, `activity_log`, `interactions`, `reminders` (per PRD §5 — build all four tables now, even though reminders UI comes in Phase 2)
- [ ] Status enum in `shared/` — single source of truth for DB, API, and UI
- [ ] First migration generated and applied; seed script with ~10 fake applications
- **Done when:** you can inspect the seeded DB and re-run migrations from scratch.
- **Learning focus:** schema design, migrations as code.

### 1.3 API — applications CRUD

- [ ] `GET /api/applications` (list, with query params for status/priority/source/search)
- [ ] `POST /api/applications`, `GET /:id`, `PATCH /:id`, `DELETE /:id`
- [ ] Input validation (zod, shared with the client)
- [ ] **Status-change rule:** any PATCH that changes `status` writes an ActivityLog row in the same transaction — this is the invariant the whole app depends on
- [ ] Unit tests for the status-transition + logging logic
- **Done when:** you can drive the full CRUD lifecycle from an HTTP client and see log rows appear.
- **Learning focus:** REST design, transactions, shared validation.

### 1.4 UI — table view & quick-add

- [ ] Applications table: sortable columns, filter bar (status, priority, source), text search
- [ ] Quick-add form (company, role, URL, source, status) — target: under 30 seconds to add
- [ ] Inline status change from the table row
- [ ] Basic app shell: nav, routing, loading/error states
- **Done when:** the daily "add + update" loop works end to end in the browser.
- **Learning focus:** data fetching patterns, controlled forms.

### 1.5 UI — detail page & timeline

- [ ] Detail page: all fields editable, notes with markdown rendering
- [ ] Activity timeline rendered from ActivityLog (status changes, with dates)
- [ ] Interactions: add a dated, typed note (screen/interview/email) to an application
- [ ] Delete with confirmation
- **Done when:** you can reconstruct an application's full history from its detail page.

### 1.6 Migrate off the spreadsheet

- [ ] One-off import script (CSV → API or direct DB insert) for your existing data
- [ ] Sanity-check counts and statuses; archive the spreadsheet
- **Done when:** the app is your single source of truth. **Phase 1 complete.**

---

## Phase 2 — Daily Loop

**Goal:** You open the app every morning instead of your notes.
**Exit criteria:** Kanban + Today view + reminders + auto-nudges all working; auto-ghost live.

### 2.1 Kanban board

- [ ] Board view grouped by status, drag-to-change-status (writes through the same PATCH endpoint — ActivityLog invariant holds automatically)
- [ ] Card shows company, role, priority, days-in-stage
- **Done when:** dragging a card updates status and the timeline reflects it.
- **Learning focus:** drag-and-drop, optimistic UI updates.

### 2.2 Manual reminders

- [ ] Reminder CRUD API + set-a-reminder UI on the detail page (due date + message)
- [ ] Mark done / dismiss
- **Done when:** you can set and resolve a reminder on any application.

### 2.3 Scheduled job: nudges & auto-ghost

- [ ] Daily in-process scheduler (e.g., node-cron) with an idempotent run (safe to trigger twice)
- [ ] Staleness rules from PRD §6.2 create auto reminders (dedupe: don't re-nudge an application that already has a pending auto-nudge)
- [ ] Auto-ghost per PRD §5: 30d from Applied, 21d from Screening/Interviewing → status `Ghosted` via the normal transition path (logged)
- [ ] Self-healing: any activity on a ghosted application restores its previous stage (derive "previous" from ActivityLog)
- [ ] Manual "run now" endpoint for testing; unit tests for rule logic
- **Done when:** seeding stale data and running the job produces the right nudges and ghosts, and touching a ghosted app un-ghosts it.
- **Learning focus:** scheduled jobs, idempotency, rule engines in miniature.

### 2.4 Today view (new landing page)

- [ ] Sections: overdue reminders, due today, recently active applications
- [ ] One-click actions: done/dismiss/snooze reminder, jump to application
- [ ] Settings page: nudge and ghost thresholds (configurable per PRD)
- **Done when:** your morning check-in takes under two minutes. **Phase 2 complete.**

---

## Phase 3 — Insight

**Goal:** Answer "what's working?" from a dashboard.
**Exit criteria:** Funnel, response rate by source, time-in-stage, and weekly trend all render from real data.

### 3.1 Analytics queries

- [ ] SQL/Drizzle queries over ActivityLog: funnel counts + stage-to-stage conversion, response rate (overall & by source), median time-in-stage, applications per week, ghost rate by source
- [ ] Expose as one `/api/analytics` endpoint (or a few small ones)
- **Done when:** endpoint returns correct numbers for a hand-verified fixture dataset.
- **Learning focus:** aggregate SQL, deriving metrics from event logs.

### 3.2 Dashboard UI

- [ ] Chart library (e.g., Recharts): funnel, bar-by-source, weekly trend line
- [ ] Date-range filter
- **Done when:** you can answer the PRD's three funnel questions without manual counting.

### 3.3 Export & optional extras

- [ ] CSV export of applications + activity log
- [ ] _(Optional)_ email digest of the Today view
- [ ] _(Optional — only if the pain arrived)_ Contact entity, per PRD §11 promotion trigger
- **Done when:** data is portable. **Phase 3 complete.**

---

## Phase 4 — Extras (earn their way in)

Deliberately unscoped. Candidates, in rough order of likely value:

- [ ] Paste-a-URL helper: fetch a job posting URL, parse title/company to prefill quick-add
- [ ] Indeed integration (API-first, no scraping)
- [ ] Hosting hardening: deploy to a VPS/Fly.io, password gate, backups
- [ ] Anything else that survives the test: _"Will I use this weekly?"_

**Rule from the PRD:** each Phase 4 item requires a one-paragraph written justification to yourself before you build it.

---

## Decision & Learning Log

Append entries as you go — this becomes your project retrospective for free.

| Date | Task | Decision / lesson |
| ---- | ---- | ----------------- |
| 2026-07-04 | 1.1 | Chose Hono over Express — better native TS ergonomics. Kept ESLint configs separate (client vs server/shared) since React-specific rules don't apply to the server. Prettier v3 respects .gitignore so no .prettierignore needed. |
