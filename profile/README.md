# PXL Automation

PXL Automation is a centralized, AI-assisted operations platform for a digital marketing agency. It reduces repetitive work, centralizes client operations, gives content workflows visibility, standardizes reporting, and helps the team scale delivery — without replacing human creativity, strategy, or client relationships.

## What This Means In Plain English

PXL Automation is the agency's shared operations hub. Instead of client details, files, content tasks, approvals, reminders, and reports living scattered across chat, spreadsheets, folders, and separate apps, the system keeps the important information connected in one workflow.

It helps with: lead capture, client onboarding, content planning and production, caption / hashtag / reel-script drafting, client approvals and revision tracking, scheduling and **publishing to Facebook & Instagram**, analytics and AI insights, report preparation, file organization, an in-portal Google Drive browser, and the background automations that tie it all together.

## Core Principle

```text
Automation prepares, routes, records, reminds, drafts, and publishes on schedule.
Humans decide, create, review, approve, and build relationships.
```

The platform removes repetitive operations *around* the team so people can focus on strategy, creativity, client communication, and final decisions.

## Glossary For Non-Technical Teams

| Term | Simple Meaning |
| --- | --- |
| Portal (`pxl-portal`) | The website/dashboard people log in to use |
| Backend / API (`pxl-api`) | The rule keeper behind the scenes that saves data, runs automation, and talks to other tools |
| Database | The official filing cabinet (Neon PostgreSQL) for all records — the single source of truth |
| AI | A drafting assistant for captions, briefs, scripts, hashtags, tags, and performance summaries |
| Automation | Background tasks the backend runs by itself after an event |
| Automation log | A row written every time an automated step happens, with its status and any error |
| Social connection | A client's authorized Facebook Page / Instagram account the API can publish to |
| Asset | A file such as a logo, graphic, video, reel, report, or brand document |
| Role-based access | Different users (`ADMIN`, `TEAM`, `CLIENT`) see different things |

## Repository Structure

| Folder | Purpose | Stack |
| --- | --- | --- |
| `pxl-api` | Backend API — data access, business rules, auth, AI calls, social publishing, in-process automation | NestJS 11, TypeScript, Drizzle ORM, PostgreSQL (Neon), JWT |
| `pxl-portal` | Public marketing site, admin portal, and client portal | Next.js 16, React 19, TypeScript, Tailwind v4, TanStack Query/Table |
| `pxl-n8n-workflows` | **Archived.** Original n8n workflow JSON, kept for reference only — all automation now runs inside `pxl-api` | n8n JSON (archived) |
| `docs` | Planning, standards, and architecture documentation | Markdown |

## System Architecture

Plain-English version:

- People use the **portal**. The portal only renders screens and sends API requests — no secrets or business rules live there.
- The portal asks the **backend** to save or load information.
- The backend checks the rules (auth, roles, validation) and reads/writes the **database**, the official record.
- When a business event happens (client created, content scheduled, lead submitted), the backend runs its **automation** in-process: it provisions Drive folders, creates Calendar reminders, emails the team, and auto-publishes approved content to Meta — recording each step in the automation log.
- **AI** is only ever called from the backend, never from the browser.

```mermaid
flowchart TB
  Public["Public visitors / leads"] --> Portal
  Client["Client users"] --> Portal
  Team["Admin / team users"] --> Portal

  Portal["pxl-portal<br/>Next.js 16 + React 19 + Tailwind v4<br/>Hosted on Vercel"]
  Portal --> API["pxl-api<br/>NestJS 11 + Drizzle + JWT<br/>Hosted on Render / Railway"]

  API --> DB["Neon PostgreSQL<br/>Single source of truth"]
  API --> AI["AI layer<br/>Groq / OpenAI-compatible"]
  API --> Sched["Scheduler<br/>NestJS cron jobs"]
  API --> Logs["automation_logs"]

  Sched --> Drive["Google Drive<br/>Client workspace folders"]
  Sched --> Calendar["Google Calendar<br/>Publishing reminders"]
  Sched --> Email["SMTP<br/>Team notifications"]
  Sched --> Meta["Meta Graph API<br/>Auto-publish + insights"]
  Meta --> DB
```

### Tech Stack

**Backend (`pxl-api`)** — NestJS 11, TypeScript, Drizzle ORM over `node-postgres` (Neon-compatible), JWT auth (`passport-jwt`) with role guards, `helmet`, `@nestjs/throttler` rate limiting, `@nestjs/schedule` cron, `@nestjs/swagger` API docs, `googleapis` (Drive + Calendar), `nodemailer` (SMTP), `bcryptjs`, AES-encrypted social tokens, `zod` config validation, `class-validator` / `class-transformer`.

**Frontend (`pxl-portal`)** — Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS v4, TanStack React Query + React Table, Axios, Radix UI / shadcn/ui, Recharts, dnd-kit, `sonner`, `next-themes`, Zod. Auth state is handled client-side by an auth gate; every endpoint is still enforced on the backend.

**Hosting / services** — Vercel (portal), Render or Railway (API), Neon (PostgreSQL), Google Drive + Calendar, Meta Graph API, SMTP, Groq/OpenAI-compatible AI. GitHub Actions for CI.

### Ownership Rules

- Neon PostgreSQL is the single source of truth.
- The backend owns all business logic, permissions, and validation.
- The frontend only displays screens and sends API requests.
- AI keys and social tokens never reach the frontend; social tokens are stored encrypted.
- All automation runs in-process in the backend — there is no external workflow tool to host.
- Google Drive stores files only; Google Calendar holds reminders only.
- Social platforms are publishing destinations and metric sources, not the operational source of truth.

## Roles

| Role | Description |
| --- | --- |
| `ADMIN` | Full system access and operational control; the only role that can create users |
| `TEAM` | Internal production, strategy, approval, reporting, and client management access |
| `CLIENT` | Client-facing access to their own approvals, assets, reports, files, and dashboard |

## Backend: `pxl-api`

NestJS application exposing a REST API under the global prefix `/api`. Interactive Swagger docs are served at `/api/docs` outside production.

### Modules

`auth` · `users` · `clients` · `client-portal` · `leads` · `onboarding` · `onboarding-tasks` · `campaigns` · `content` · `content-pillars` · `content-templates` · `approvals` · `assets` · `analytics` · `reports` · `ai` · `assistant` · `automation` (+ `automation-retry`) · `scheduler` · `calendar` · `drive` · `social-connections` · `notifications` · `health`

### API Reference

All routes below are prefixed with `/api`. Unless marked **public**, they require a JWT; most are restricted to `ADMIN`/`TEAM`, and the client-portal routes to `CLIENT`.

| Area | Key endpoints |
| --- | --- |
| Auth | `POST /auth/login` (public) · `POST /auth/register` (ADMIN) · `GET /auth/me` |
| Clients | `GET/POST /clients` · `GET/PATCH /clients/:id` |
| Leads | `POST /leads` (public) · `GET /leads` · `GET/PATCH /leads/:id` · `POST /leads/:id/convert` |
| Onboarding | `POST /onboarding` (public intake → `ONBOARDING` client) |
| Onboarding tasks | `GET/PATCH /onboarding-tasks` (per-client checklist) |
| Campaigns | `GET/POST /campaigns` · `GET/PATCH/DELETE /campaigns/:id` |
| Content | `GET/POST /content` · `GET/PATCH /content/:id` · `PATCH /content/:id/schedule` · `PATCH /content/:id/publish` |
| Content pillars | `GET/POST/PATCH/DELETE /content-pillars` |
| Content templates | `GET/POST/PATCH/DELETE /content-templates` |
| Approvals | `GET/POST /approvals` · `PATCH /approvals/:id` · approval comment threads |
| Assets | `GET/POST /assets` · `GET/PATCH /assets/:id` · `POST /assets/:id/auto-tag` |
| Analytics | `GET/POST /analytics` · `GET /analytics/best-times` · `GET /analytics/content/:contentItemId` |
| Reports | `GET/POST /reports` · `GET/PATCH /reports/:id` |
| Automation | `GET /automation/logs` (`?status=FAILED` filter) · `POST /automation/logs/:id/retry` |
| Assistant | `POST /assistant/chat` (public lead-gen chatbot) |
| AI | `POST /ai/generate-caption` · `/generate-hashtags` · `/generate-reel-script` · `/generate-brief` · `/generate-broll` · `/generate-overlay` · `/generate-tags` · `/generate-template` · `/analyze-performance` |
| Google Drive | `/drive/clients/:clientId/items` · `/folders` · `/files` (upload/download/delete) |
| Social connections | `/clients/:clientId/social-connections` (list / Meta OAuth URL / sync / disconnect) · `GET /social-connections/meta/callback` (public OAuth return) |
| Client portal | `/client-portal/overview` · `/me` · `/content` · `/approvals` (+ `/:id`, `/:id/comments`) · `/assets` · `/reports` · `/client-portal/drive/*` |
| Health | `GET /health` |

> AI generation endpoints accept `language` (`EN` | `TAGLISH`) and `seo` (boolean) options. When no provider key is configured, the API returns deterministic fallback drafts so the workflow stays testable.

## Frontend: `pxl-portal`

### Public pages

| Route | Purpose |
| --- | --- |
| `/` | Public PXL Digital marketing landing page (hero, services, process, results, FAQ, contact) with a floating chat widget |
| `/get-started` | Guided "free growth plan" funnel that captures a lead through a short multi-step flow |
| `/onboarding` | Public client onboarding intake → creates an `ONBOARDING` client and triggers Drive provisioning |
| `/login` | Email + password login; redirects by role |
| `/meta-connected` | Landing page the Meta OAuth flow returns to (success / error) |

### Admin & team pages

`/admin/dashboard` · `/admin/clients` · `/admin/clients/[id]` · `/admin/content` · `/admin/content/[id]` · `/admin/content-templates` · `/admin/calendar` · `/admin/campaigns` · `/admin/campaigns/[id]` · `/admin/approvals` · `/admin/analytics` · `/admin/reports` · `/admin/leads` · `/admin/assets` · `/admin/automation` · `/admin/users` *(ADMIN only)*

### Client pages

`/client/dashboard` (status, content, approvals + comments, assets, reports) · `/client/files` (in-portal Drive browser for the client's own folder)

### Login redirects

| Role | Redirect |
| --- | --- |
| `ADMIN` / `TEAM` | `/admin/dashboard` |
| `CLIENT` | `/client/dashboard` |

A `CLIENT` user's login email must match a client record's email for the dashboard to resolve their workspace.

## Database

Neon PostgreSQL, modeled with Drizzle ORM (`pxl-api/src/database/schema.ts`).

### Tables

| Table | Holds |
| --- | --- |
| `users` | Login accounts and roles |
| `clients` | Client profiles, status, Drive folder URL |
| `leads` | Public inquiries with heuristic `score` / `score_band` (COLD/WARM/HOT) |
| `campaigns` | Client campaigns (goal, budget, audience, offer, dates, status) |
| `content_items` | Content with status, caption, hashtags, target `platforms` / `social_targets`, publish results |
| `approvals` | Approval state, feedback, revision count |
| `approval_comments` | Threaded comments on an approval (team ↔ client) |
| `assets` | Client/content-linked Drive file records with type, version, tags |
| `analytics` | Per-content performance metrics |
| `reports` | Client report records with period, summary, Drive link |
| `onboarding_tasks` | Standard onboarding checklist auto-seeded per client |
| `content_pillars` | Recurring content themes with monthly cadence |
| `content_templates` | Reusable content skeletons (per-client or agency-wide) |
| `social_connections` | Connected Facebook Pages / Instagram accounts (encrypted page tokens) |
| `meta_authorizations` | Per-client Meta owner authorizations (encrypted user tokens) |
| `meta_oauth_states` | One-time OAuth nonces for the Meta connect flow |
| `automation_logs` | Every automation event, its status, and any error |

### Content Lifecycle

```mermaid
stateDiagram-v2
  [*] --> IDEA
  IDEA --> DRAFTING: Strategy and brief
  DRAFTING --> DESIGNING: Production starts
  DESIGNING --> INTERNAL_REVIEW: Draft ready
  INTERNAL_REVIEW --> CLIENT_APPROVAL: Team sends for client review
  CLIENT_APPROVAL --> APPROVED: Client approves
  CLIENT_APPROVAL --> REVISION_REQUESTED: Client requests edits
  REVISION_REQUESTED --> DRAFTING: Team revises
  APPROVED --> SCHEDULED: Publishing date set
  SCHEDULED --> PUBLISHED: Auto-published to Meta (or marked published)
  PUBLISHED --> REPORTED: Analytics and report complete
  REPORTED --> IDEA: Insights feed next cycle
```

## Automation (in-process)

The backend runs all repeated operational work itself, right after a business event or on a cron schedule — there is no external tool to host or point webhooks at. Every event is written to `automation_logs` and visible at `GET /api/automation/logs`.

### Events

`client-created` · `drive-folder-provisioned` · `lead-created` · `content-scheduled` · `content-calendar-reminder` · `content-auto-published` · `content-auto-publish-failed` · `content-auto-publish-abandoned` · `approval-reminder` · `lead-follow-up-reminder` · `analytics-ingested`

### What runs when

- **Client created** (manually or via onboarding/lead conversion) → create the Google Drive folder with standard subfolders, seed the onboarding checklist, email the team (`client-created`, `drive-folder-provisioned`).
- **Content scheduled** → create a Google Calendar reminder (`content-scheduled`, `content-calendar-reminder`).
- **Scheduled time reached** → a minute-level cron auto-publishes approved content to its selected Meta platforms (`content-auto-published`). Permanent failures stop after 3 attempts (`content-auto-publish-failed` → `content-auto-publish-abandoned`).
- **Hourly** → pull each published post's Meta insights into the `analytics` table (`analytics-ingested`).
- **Daily-ish reminders** → approvals pending >24h (`approval-reminder`) and leads left `NEW` >24h (`lead-follow-up-reminder`), each re-flagged at most once per day.
- **Failed automations** (Drive provisioning, calendar reminder, etc.) can be re-run from the logs via `POST /api/automation/logs/:id/retry`.

## Social Publishing (Meta)

PXL publishes directly to clients' Facebook Pages and Instagram accounts through the Meta Graph API.

1. From a client's detail page, an admin/team user starts the Meta connect flow (`POST /api/clients/:clientId/social-connections/meta/oauth-url`).
2. The Page owner authorizes via Meta; Meta redirects back to `GET /api/social-connections/meta/callback`, which exchanges the code, stores the connection (tokens **encrypted** with `SOCIAL_TOKEN_ENCRYPTION_KEY`), and returns the browser to `/meta-connected`.
3. Content items carry `platforms` / `socialTargets`. Approved, scheduled content is auto-published to those targets at its scheduled time; results are saved on the content item and logged.
4. Published-post insights are ingested hourly into analytics.

## Google Drive

Google Drive stores files only; the API stores the Drive links and references. When a client is created, the API provisions:

```text
PXL Clients/
  Client Name/
    01 Brand Assets/
    02 Monthly Content/
    03 Reels/
    04 Graphics/
    05 Approved/
    06 Published/
    07 Reports/
```

The portal also includes a **protected Drive browser**: admins/team browse, upload, create folders, download, and delete inside a client's folder; clients use `/client/files` for their own folder only. Google credentials stay in the API, and every request is checked against the client's saved Drive root.

If Drive credentials or the parent folder ID are missing, provisioning is skipped silently and the folder URL can be set by hand on the client record.

## Main Connected Workflow

```mermaid
flowchart LR
  Lead["Lead form / get-started"] --> Leads["Leads"]
  Leads --> Qualify["Human: qualify"]
  Qualify --> Onboarding["Onboarding"]
  Onboarding --> Clients["Clients + Drive folder"]
  Clients --> Strategy["Human: strategy + campaigns"]
  Strategy --> Briefs["AI: briefs / ideas"]
  Briefs --> Production["Human: production"]
  Production --> Drafts["AI: captions / hashtags / reels"]
  Drafts --> Review["Human: internal review"]
  Review --> Approval["Client approval"]
  Approval -->|Approved| Schedule["Schedule"]
  Approval -->|Revision| Production
  Schedule --> Publish["Auto-publish to Meta"]
  Publish --> Analytics["Insights ingested"]
  Analytics --> Reports["AI-assisted reports"]
  Reports --> Strategy
```

### Human vs Automation (summary)

| Area | Human work | Automation work |
| --- | --- | --- |
| Lead gen | Qualify, contact, close | Capture lead, score it, notify, follow-up reminder |
| Onboarding | Clarify details, set priorities | Save client, create Drive folders, seed checklist, notify team |
| Strategy | Pillars, offers, campaign direction | Draft ideas and first-pass briefs |
| Production | Design, edit, polish | Track status, store asset links/versions |
| Captions / reels | Tone, brand voice, final cut | Generate captions, hashtags, scripts, B-roll, overlays (EN/Taglish, SEO) |
| Approvals | Review, decide, interpret feedback | Route, save feedback, count revisions, remind, thread comments |
| Publishing | Final check | Calendar reminder, auto-publish to Meta, update status |
| Analytics | Interpret results | Ingest Meta insights, best-time suggestions, AI summaries |
| Reporting | Final narrative | Draft report, store Drive URL, notify |

### Workflow-study mapping

The system implements the 20-part PXL workflow study. Highlights of how it maps to features:

| Study area | Where it lives now |
| --- | --- |
| #3 Client onboarding | `/onboarding`, `clients`, Drive provisioning, onboarding checklist |
| #4 Content strategy | `campaigns`, `content_pillars`, AI briefs |
| #5 Content production | `content_items`, `content_templates`, asset tracking |
| #6 Reels & video | AI reel scripts, B-roll, overlay/auto-caption |
| #7 Captions | AI captions/hashtags with Taglish + SEO modes |
| #8 Approvals | `approvals` + comment threads, reminders |
| #9 Publishing | scheduling, Calendar reminders, Meta auto-publish, best-times |
| #10 Analytics | metrics, hourly Meta ingestion, AI performance insights |
| #11 Lead generation | `/get-started`, lead scoring, public chat assistant, follow-up reminders |
| #12 File management | Google Drive folders + in-portal browser, asset auto-tagging |

## Environment Setup

### API — `pxl-api/.env` (see `pxl-api/.env.example`)

```env
# Server
NODE_ENV=development
PORT=4000
CORS_ORIGIN=http://localhost:3000

# Database / auth
DATABASE_URL=postgresql://user:password@host:5432/pxl_automation
JWT_SECRET=replace-with-a-long-random-secret
JWT_EXPIRES_IN=7d

# AI (Groq / OpenAI-compatible)
AI_PROVIDER=groq
AI_MODEL=
GROQ_API_KEY=
OPENAI_API_KEY=

# Meta publishing
META_GRAPH_API_VERSION=v25.0
META_APP_ID=
META_APP_SECRET=
META_LOGIN_CONFIG_ID=
META_OAUTH_REDIRECT_URI=http://localhost:4000/api/social-connections/meta/callback
FRONTEND_URL=http://localhost:3000
SOCIAL_TOKEN_ENCRYPTION_KEY=   # node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"

# Google Drive (parent folder for auto-created client folders)
DRIVE_CLIENTS_PARENT_FOLDER_ID=
# Service account:
GOOGLE_DRIVE_CLIENT_EMAIL=
GOOGLE_DRIVE_PRIVATE_KEY=
# …or OAuth (personal My Drive):
GOOGLE_DRIVE_CLIENT_ID=
GOOGLE_DRIVE_CLIENT_SECRET=
GOOGLE_DRIVE_REFRESH_TOKEN=

# Google Calendar (reuses Drive credentials; defaults to "primary")
GOOGLE_CALENDAR_ID=

# SMTP team notifications (blank to disable)
SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=
SMTP_FROM=PXL <noreply@pxl.local>
TEAM_NOTIFICATION_EMAIL=

# Local admin seed
SEED_ADMIN_EMAIL=admin@pxl.local
SEED_ADMIN_NAME=PXL Admin
SEED_ADMIN_PASSWORD=change-this-password
```

### Portal — `pxl-portal/.env` (see `pxl-portal/.env.example`)

```env
# Base URL of pxl-api (the client appends /api itself). Only NEXT_PUBLIC_* is exposed.
NEXT_PUBLIC_API_URL=http://localhost:4000
```

Restart the affected app after changing its `.env`.

## Local Commands

This is a pnpm workspace; run `pnpm install` in each app first.

### Backend (`pxl-api`)

```bash
pnpm run db:generate    # generate SQL from the Drizzle schema
pnpm run db:migrate     # apply migrations
pnpm run seed:admin     # create the seed admin account
pnpm run start:dev      # watch mode
pnpm run start          # run built server
pnpm run typecheck
pnpm run lint
pnpm run test
pnpm run build
pnpm run db:studio      # Drizzle Studio
```

### Portal (`pxl-portal`)

```bash
pnpm run dev
pnpm run typecheck
pnpm run lint
pnpm run test
pnpm run e2e            # Playwright end-to-end tests
pnpm run build
```

## Deployment Targets

| Layer | Recommended host |
| --- | --- |
| Frontend | Vercel |
| Backend | Render or Railway |
| Database | Neon PostgreSQL |
| Automation | In-process (runs inside the backend) |
| Files | Google Drive |
| Publishing | Meta Graph API |

## End-to-End Flow (MVP)

1. A visitor submits the `/get-started` funnel (or asks the public chat assistant) → a lead is saved and scored, and the team is notified.
2. A qualified lead is converted to a client (or the client self-submits `/onboarding`).
3. The API saves the client, provisions the Drive folder + subfolders, seeds the onboarding checklist, and emails the team.
4. An admin creates the client's login account (matching email) and connects their Meta Page.
5. The team plans campaigns and content; AI assists with briefs, captions, hashtags, and reel scripts.
6. Content goes through internal review → client approval (with comment threads) → approve or revise.
7. Approved content is scheduled; the API creates a Calendar reminder and auto-publishes to Meta at the scheduled time.
8. Post insights are ingested hourly into analytics; AI drafts performance summaries and best-time suggestions.
9. The team finalizes reports; the client views approvals, assets, files, and reports in their portal.
10. Insights feed the next cycle of strategy.

## MVP Limitations & Future Work

Humans still finalize strategy, edit AI drafts, and own client communication. Known future work includes video auto-captioning/clipping, cross-platform publishing beyond Meta (TikTok, LinkedIn, YouTube), richer report PDF generation, and validating Meta insight metric mappings against the live Graph API version.

## Strategic Direction

PXL Automation should remain a centralized operations layer: help the team move faster, reduce duplicated work, standardize delivery, and make reporting easier — while keeping strategy, taste, client judgment, and creative quality firmly in human hands.
