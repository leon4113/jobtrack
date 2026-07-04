# PRD: Job Application Tracker (Personal)

**Version:** 1.1 · **Date:** July 2026 · **Owner:** You (solo developer)
**Status:** Approved — open questions resolved, ready for Phase 1

---

## 1. Overview

A personal web application for tracking job applications through their full lifecycle — from "found an interesting posting" to "offer" or "rejected." The tool replaces ad-hoc spreadsheets with a purpose-built system that also nudges you to follow up at the right time and shows you which parts of your job search are working through funnel analytics.

This is a single-user tool built for personal use. Every scoping decision below favors simplicity and shippability over generality: no multi-tenancy, no team features, no admin panels.

## 2. Goals

1. **Never lose track of an application.** Every application has a clear status, history, and next action at all times.
2. **Never miss a follow-up.** The system proactively reminds you when an application has gone quiet or an action is due.
3. **Learn from the data.** Understand response rates, where applications stall, and which sources/roles convert best.
4. **Be a great learning project.** Built solo with Claude Code as a pair-programming mentor, so the architecture should exercise real full-stack skills without drowning in incidental complexity.

## 3. Non-Goals (v1)

- Multi-user support, authentication beyond a single-user login, or sharing features.
- Automatic scraping or importing of job postings (job board integration is deferred — see Phase 4).
- Resume/cover letter version management.
- Mobile native app (the web app should be responsive, but that's it).
- Email integration (reading your inbox to auto-update statuses).

## 4. User & Context

One user: you, an active job seeker. Typical usage pattern:

- **Daily (2–10 min):** Add newly found/applied jobs, update statuses after emails or calls, check today's reminders.
- **Weekly (10–15 min):** Review the pipeline, act on stale applications, glance at analytics.

The app must make the daily loop _fast_ — adding an application should take under 30 seconds, and updating a status should take one or two clicks.

## 5. Core Concepts & Data Model

### Application

The central entity. Key fields:

| Field                   | Type        | Notes                                          |
| ----------------------- | ----------- | ---------------------------------------------- |
| id                      | uuid        |                                                |
| company                 | string      | required                                       |
| role_title              | string      | required                                       |
| job_url                 | string      | optional                                       |
| source                  | enum/string | e.g., Indeed, LinkedIn, referral, company site |
| location                | string      | optional; include remote/hybrid/onsite flag    |
| salary_range            | string      | optional, free text is fine for v1             |
| status                  | enum        | see status model below                         |
| priority                | enum        | high / medium / low                            |
| date_found              | date        |                                                |
| date_applied            | date        | nullable until applied                         |
| notes                   | text        | markdown supported                             |
| created_at / updated_at | timestamp   |                                                |

### Status model (the funnel)

`Wishlist → Applied → Screening → Interviewing → Offer → Accepted`
Terminal branches from any stage: `Rejected`, `Withdrawn`, `Ghosted`.

**Auto-ghosting (decided):** applications transition to `Ghosted` automatically after X days of no activity — default 30 days from `Applied`, 21 days from `Screening`/`Interviewing` (both configurable). Three rules govern this:

1. **Self-healing:** any new activity (status change, note, interaction) on a ghosted application automatically restores its previous stage.
2. **Configurable thresholds** with the defaults above, tunable in settings.
3. **Logged like any transition:** ghost events flow through ActivityLog, enabling ghost-rate-by-source analytics for free.

Runs in the same daily scheduled job as the staleness nudges.

Every status change is recorded in an **ActivityLog** (application_id, from_status, to_status, timestamp, optional note). This log is what powers both the timeline view and the analytics — do not skip it.

### Interaction

Optional child records of an application: interviews, phone screens, emails sent. Fields: type, date, notes, outcome. Keep this lightweight in v1 — a typed note with a date is enough.

### Reminder

| Field          | Type   | Notes                            |
| -------------- | ------ | -------------------------------- |
| id             | uuid   |                                  |
| application_id | fk     |                                  |
| due_date       | date   |                                  |
| message        | string | e.g., "Follow up with recruiter" |
| type           | enum   | manual / auto                    |
| status         | enum   | pending / done / dismissed       |

## 6. Features

### 6.1 Application CRUD & Pipeline View (P0)

- Quick-add form (company, role, URL, source, status) — optimized for speed.
- Kanban-style board grouped by status with drag-to-change-status, plus a sortable/filterable table view.
- Detail page per application: all fields, interaction history, activity timeline, reminders.
- Search and filters (status, priority, source, date range).

### 6.2 Reminders & Follow-up Nudges (P0)

- **Manual reminders:** set a date + message on any application.
- **Auto-nudges (staleness rules):** configurable defaults, e.g.:
  - Applied but no response in 10 days → "Consider following up."
  - Interviewing with no activity in 7 days → "Check in with your contact."
  - Wishlist item untouched for 14 days → "Apply or archive?"
- A **Today view** (default landing page): reminders due today/overdue + recently active applications.
- Delivery in v1 is _in-app only_. Email/push notifications are a Phase 3 enhancement — don't build notification infrastructure before the core loop works.

### 6.3 Analytics (P1)

- **Funnel chart:** count of applications reaching each stage; conversion rate between stages (e.g., Applied → Screening %).
- **Response rate:** % of applications that got any response, overall and by source.
- **Time-in-stage:** median days spent in each stage (derived from ActivityLog).
- **Trend:** applications submitted per week.
- All computed from ActivityLog + application fields; no separate analytics store needed at this scale.

### 6.4 Job Board Integration (P3 — deferred)

- Explicitly out of MVP. When revisited: start with a "paste URL → parse title/company" helper or the Indeed API rather than scraping.
- Design impact today: keep `source` and `job_url` as first-class fields, and keep the quick-add form logic separate from the UI so an importer can reuse it later.

## 7. Roadmap & Phasing

| Phase                     | Scope                                                                      | Definition of done                                               |
| ------------------------- | -------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| **1 — Core tracker**      | Data model, CRUD, table view, detail page, status changes with ActivityLog | You've fully replaced your spreadsheet                           |
| **2 — Daily loop**        | Kanban board, Today view, manual reminders, auto-nudge rules               | You open the app every morning instead of your notes             |
| **3 — Insight**           | Analytics dashboard, CSV export, optional email digest                     | You can answer "what's my response rate on Indeed vs referrals?" |
| **4 — Extras (optional)** | Job board integration, URL parsing, anything else that earns its way in    | Only if the pain justifies it                                    |

Ship Phase 1 end-to-end before touching Phase 2. A working ugly tracker beats a half-built beautiful one.

## 8. Technical Recommendations

**Stack (decided): TypeScript end-to-end.**

- **Frontend:** React + Vite. Client-side SPA talking to a JSON API.
- **Backend:** Node with a lightweight framework (Express or Hono).
- **Database:** SQLite via Drizzle ORM, with Drizzle's migration tooling from day one.
- **Shared types:** application status enums, DTOs, and nudge-rule configs live in a shared package/folder so frontend and backend can't drift.

General shape:

- **Architecture:** classic three-layer web app. Frontend SPA + REST/JSON API + relational database. Avoid microservices, queues, or anything distributed — this is a single-user CRUD app.
- **Database sizing:** SQLite is genuinely sufficient (single user, low write volume) and removes ops burden; revisit Postgres only if hosting requirements demand it.
- **Reminders/nudges:** a scheduled job (cron or in-process scheduler) that evaluates staleness rules once a day and creates Reminder rows. Resist the urge to build a real-time notification system.
- **Analytics:** SQL queries over ActivityLog rendered with a chart library. No analytics pipeline.
- **Hosting:** localhost is a valid v1. If hosting: a single small VPS or a platform like Fly.io/Railway. Add a simple password gate if it's internet-exposed.
- **Testing:** unit-test the status-transition logic and the staleness rules (the two places bugs actually hurt); don't chase coverage elsewhere.
- **Pair-programming workflow:** work feature-by-feature per the roadmap. Good Claude Code sessions map 1:1 to rows in the phase table above.

## 9. Success Metrics

Since this is personal, metrics are honest self-checks:

1. You stop using the old spreadsheet within 2 weeks of Phase 1 shipping.
2. Zero missed follow-ups after Phase 2 (every stale application gets a nudge).
3. You can answer three funnel questions from the dashboard without manual counting after Phase 3.
4. Adding an application takes < 30 seconds; updating a status takes ≤ 2 clicks.

## 10. Risks & Mitigations

| Risk                                             | Mitigation                                                                            |
| ------------------------------------------------ | ------------------------------------------------------------------------------------- |
| Scope creep (the classic solo-project killer)    | Phases are gated; Phase 4 requires justification in writing to yourself               |
| Over-engineering the stack to "learn everything" | One deliberate learning goal per phase; everything else stays boring                  |
| Nudge fatigue → ignoring reminders               | Rules are configurable and dismissible; tune thresholds after 2 weeks of real use     |
| Abandonment after job search ends                | Acceptable outcome — but ActivityLog + CSV export preserves the data and the learning |

## 11. Resolved Decisions

1. **Contacts:** notes field only in v1. Promotion trigger: if you find yourself repeatedly searching notes for a person's name, add a Contact entity in Phase 3. ActivityLog + notes make later migration straightforward.
2. **Ghosted:** fully automatic with self-healing restore, configurable thresholds (30d from Applied, 21d from Screening/Interviewing), logged via ActivityLog. See §5.
3. **Stack:** TypeScript end-to-end — React/Vite, Node (Express or Hono), SQLite + Drizzle. See §8.
