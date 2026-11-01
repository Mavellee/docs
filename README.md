# Iceberg – Engineering Docs

Iceberg is a dating web app (mobile-first, installable as a PWA) built around three ideas:
**intent-based matching**, **optional astrologer review of a match**, and an **AI companion ("Iceberg")**
that keeps conversations going. This folder is the single source of truth for *what* we are building,
*how* it fits together, and *in what order* we build it.

> Read these in order the first time. After that, use them as a reference.

| # | Doc | Answers |
|---|-----|---------|
| 01 | [Product and scope](01-product-and-scope.md) | What are we building? What is in the MVP and what is not? Which screen maps to which feature? |
| 02 | [Architecture](02-architecture.md) | How do the pieces fit together? Which layer does what? Why these decisions? |
| 03 | [Tech stack explained](03-tech-stack-explained.md) | What is each tool, why did we pick it, and where does it appear in the code? |
| 04 | [Environments and deployment](04-environments-and-deployment.md) | How do local → staging → production work? How does the *same* commit reach production? |
| 05 | [Backend guide](05-backend-guide.md) | Folder layout, layering rules, API conventions, full endpoint list, how to add an endpoint |
| 06 | [Frontend guide](06-frontend-guide.md) | Design system, route map, screen-by-screen build spec, state and data rules |
| 07 | [User flows](07-user-flows.md) | Sequence diagrams of every important flow (sign-up, like → match → review → chat, events…) |
| 08 | [Security and privacy](08-security-and-privacy.md) | Threat model, rules we never break, DPDP-style data handling |
| 09 | [Testing strategy](09-testing-strategy.md) | What we test, with which tool, and what must pass before a merge |
| 10 | [Roadmap and tasks](10-roadmap-and-tasks.md) | Milestone-by-milestone checklist with a "definition of done" for each |
| 11 | [Local setup](11-local-setup.md) | Step-by-step: get the app running on your machine |

## Repository layout

```
Iceberg_dating_MVP/
├── db/            Postgres schema (migrations), seed data, DB scripts and SQL tests   (already built)
├── web/           Next.js app: the website AND the backend API (route handlers)
├── supabase/      (added in Milestone 1) Supabase CLI config + Edge Functions for background jobs
├── docs/          You are here
├── .github/       CI/CD workflows
└── package.json   Convenience scripts that delegate to db/ and web/
```

## Three rules that keep the project healthy

1. **The database is the source of truth.** Every schema change is a new numbered file in `db/migrations/`. We never edit a migration that has already run in staging or production.
2. **The same commit goes to staging, then production.** Nothing is ever "fixed directly on prod". See doc 04.
3. **One vertical slice at a time.** Each milestone ships something a user can click end-to-end (DB → API → UI → test) before we start the next.
