# 03 – Tech stack explained

For every tool: **what it is**, **why we use it**, **where it shows up**, and **the one thing to understand**.
Tools are grouped by when we introduce them – we add a dependency only when its milestone needs it.

Legend: ✅ installed in Milestone 0 · 🔜 added in the milestone shown

## A. Core application

### Next.js (App Router) + React + TypeScript ✅
- **What:** A React framework that also runs server code. A folder `src/app/dashboard/page.tsx` becomes the URL `/dashboard`; `src/app/api/v1/likes/route.ts` becomes `POST /api/v1/likes`.
- **Why:** One project gives us pages, API, server-side rendering (fast first load and SEO for public event pages) and one deploy on Vercel.
- **Understand this:** components are **Server Components by default** (run only on the server, can read the DB, ship no JS). Add `"use client"` only to files that need state, effects or browser APIs (swipe cards, chat input).
- **TypeScript** catches wrong field names and missing cases at build time, which matters in a codebase with 37 tables.

### Tailwind CSS + shadcn/ui ✅
- **What:** Tailwind = small utility classes (`rounded-2xl bg-primary px-4`). shadcn/ui = accessible components (Button, Dialog, Tabs, Sheet…) that are **copied into our repo** (`src/components/ui`) so we own and restyle them.
- **Why:** Fast to build, and the pink/rose Iceberg look is just design tokens (CSS variables) defined once in `globals.css`.
- **Understand this:** we never write one-off colours in components; we use tokens like `bg-primary`, `text-muted-foreground`. Changing the brand later = editing one file.

### Framer Motion ✅
- Declarative animation: swipe cards that follow the pointer, the "It's a Match!" celebration, page transitions. Used only in client components. Respect `prefers-reduced-motion`.

### TanStack Query (server state) 🔜 M0/M2
- **What:** A cache for data that lives on the server (feed, matches, messages).
- **Why:** Gives loading/error states, caching, retries, background refetch, and optimistic updates (a like feels instant) without hand-written `useEffect` fetching.
- **Rule:** *Anything that came from the API lives in TanStack Query. Never copy it into Zustand.*

### Zustand (UI state) 🔜 M3
- **What:** A tiny store for state that exists only in the browser: "is the filter sheet open", "which tab is selected", swipe-deck position.
- **Rule:** *UI state only.* If it must survive a refresh or be shared with the server, it is not Zustand's job.

### React Hook Form + Zod 🔜 M2
- **What:** RHF manages form state without re-rendering on every keystroke. **Zod** describes valid data (`z.string().max(500)`).
- **Why together:** the *same* Zod schema (in `src/shared/schemas`) validates the form in the browser **and** the request in the API. They cannot disagree.
- Zod is also used in M0 to validate **environment variables** at startup, so a missing secret fails the deploy instead of failing a user.

### Serwist (PWA) 🔜 M8
- **What:** Generates the service worker: offline shell, asset caching, push-notification handler.
- **Why:** "Add to home screen" on phones and web push for *"your match was approved"*. iOS only supports web push for installed PWAs, so the install prompt matters.

## B. Backend and data

### Route Handlers + Server Actions ✅
- **Route Handlers** (`route.ts`): the public JSON API (`/api/v1/*`). Anything a future mobile app or the astrologer console might call.
- **Server Actions**: used sparingly for simple form posts from our own pages (e.g. "save settings"). They call the **same services**, so rules live in one place.

### PostgreSQL + PostGIS (via Supabase) ✅ (schema exists in `db/`)
- **What:** The database. PostGIS adds geography types so `st_dwithin(a, b, 50000)` means "within 50 km".
- **Why:** Dating logic is relational (pairs, blocks, matches) and geographic. Postgres does both with real constraints.
- **Where:** `db/migrations/*.sql`. Business rules that must be atomic are SQL functions (doc 02).

### `pg` (node-postgres) 🔜 M0
- **What:** The driver our repositories use to run parameterised SQL.
- **Why not an ORM:** the valuable logic is already SQL functions and PostGIS queries; an ORM would fight us. Parameterised queries (`$1, $2`) make SQL injection impossible by construction.
- **Production detail:** in serverless we connect to Supabase's **pooler** (transaction mode) and keep a small pool per function instance (see `src/server/db/pool.ts`).

### Supabase (Auth, Realtime, Storage, Edge Functions, pg_cron) 🔜 M1
| Piece | Used for |
|-------|----------|
| **Auth** | Phone OTP (primary), Google, college-email magic link. Issues JWTs in cookies via `@supabase/ssr`. |
| **Realtime** | New messages (postgres changes, RLS-protected), typing + presence (broadcast/presence, never stored). |
| **Storage** | Private buckets: `photos`, `voice-intros`, `chat-media`. Always accessed via short-lived **signed URLs**. |
| **Edge Functions + pg_cron** | Background jobs: review-deadline reminders, Iceberg prompt timing, waitlist promotion, daily `purge_deleted_accounts()`. |
| **RLS** | Row Level Security – the second wall that protects rows even if API code has a bug. |

### Upstash Redis 🔜 M1
- Serverless Redis over HTTP. Used for **rate limiting** (OTP requests, likes, messages, AI calls) with a sliding-window limiter. Without it, one script could burn our SMS budget in minutes.

## C. Integrations (all behind interfaces in `src/server/integrations/`)

| Integration | Purpose | Notes |
|-------------|---------|-------|
| MSG91 / Twilio | OTP SMS | Configured **inside Supabase Auth**. Indian SMS needs **DLT registration** – start now, it takes days. Locally we use Supabase "test OTPs", no SMS sent. |
| Resend | Transactional email (welcome, review updates) | Domain verification (SPF/DKIM) needed once. |
| LLM (Claude / OpenAI) | Iceberg levels 3–4, text moderation | Key stays server-side. A daily budget guard and a fallback to the curated prompt library are mandatory. |
| AWS Rekognition / Sightengine | Photo moderation before a photo becomes visible | Photos start as `moderation_status = 'pending'`. |
| Web Push (VAPID) | Push notifications | Subscription stored in `auth_devices`. |
| `qrcode` + `html5-qrcode` | Event QR generate / scan | Token is `event_registrations.qr_token`, single-use. |

> **Why interfaces?** Every integration has a *real* implementation and a *fake* one (used in local dev and tests). The service calls `otp.send()`; it neither knows nor cares whether that hit Twilio. This is what makes the app testable and lets us swap vendors.

## D. Quality and operations

| Tool | Purpose | When |
|------|---------|------|
| ESLint + `tsc --noEmit` | Style and type errors | ✅ every commit (CI) |
| Vitest | Fast unit and integration tests for services, schemas, repositories | ✅ |
| pgTAP / SQL smoke tests | Prove DB rules (`db/tests/smoke_test.sql`) and later RLS rules | ✅ exists |
| Playwright | End-to-end browser tests of key journeys, run against staging | 🔜 M3 |
| GitHub Actions | CI (lint, types, tests, db tests, build) + migration deploys | ✅ |
| Vercel | Hosting, preview deploy for every PR, staging + production | ✅ |
| Sentry | Error tracking (server + browser), release-tagged by git SHA | 🔜 M1 |
| PostHog | Product analytics + feature flags | 🔜 M2 |
| pino | Structured JSON logs with a `requestId` on every line | ✅ |
| Better Stack / UptimeRobot | Pings `/api/health` and alerts us | 🔜 before launch |
| Cloudflare | DNS, CDN, bot protection (Turnstile on sign-up) | 🔜 before launch |

## E. What "production techniques" means concretely in this repo

1. **Config is validated at boot** (`src/env.ts`) – the app refuses to start with missing/invalid env vars.
2. **Every request has an id**, and errors have stable machine-readable codes.
3. **Input is parsed, not trusted** – Zod on every route; identity comes from the session, never from the body.
4. **Pagination is cursor-based** and every list endpoint has a max page size.
5. **Mutations that can be retried are idempotent** (client-generated ids, `on conflict`).
6. **Secrets never reach the browser** – only variables prefixed `NEXT_PUBLIC_` do, and that list is tiny and reviewed.
7. **The database has the last word** – constraints and SQL functions protect integrity even if code is wrong.
8. **Same commit, three environments** – only environment variables differ (doc 04).
9. **Migrations are forward-only and reviewed** like code.
10. **Everything privacy-sensitive is auditable** (`audit_log` is append-only).
