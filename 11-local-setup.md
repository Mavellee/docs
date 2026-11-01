# 11 – Local setup

Everything you need to run Iceberg on your machine. Windows / macOS / Linux all work; examples use Git Bash or PowerShell.

## Prerequisites

| Tool | Version | Check |
|------|---------|-------|
| Node.js | 22 or newer (24 is fine) | `node -v` |
| npm | comes with Node | `npm -v` |
| Docker Desktop | any recent | `docker --version` – and **start Docker Desktop** so the engine is running |
| Git | any recent | `git --version` |

## First-time setup (5 steps)

```bash
# 1. Install dependencies for db/ and web/
npm run setup

# 2. Start the local database (Postgres 16 + PostGIS, port 5433)
npm run db:up

# 3. Create the schema and load seed data into it
#    (db/.env is what the db scripts read; the Docker database is on port 5433)
#    PowerShell:  $env:DATABASE_URL="postgresql://postgres:postgres@localhost:5433/iceberg"; npm run db:setup
#    Git Bash:    DATABASE_URL=postgresql://postgres:postgres@localhost:5433/iceberg npm run db:setup
#    Or put that line in db/.env and just run: npm run db:setup

# 4. Configure the web app
cp web/.env.example web/.env.local

# 5. Run it
npm run dev          # http://localhost:3000
```

Check it works:

| URL | Expected |
|-----|----------|
| http://localhost:3000 | the Iceberg landing page |
| http://localhost:3000/api/health | `{"status":"ok",...}` |
| http://localhost:3000/api/ready | `{"status":"ready",...}` (proves the DB connection) |
| http://localhost:3000/api/v1/interests | `{"data":[{"id":"…","name":"Travel",…}]}` |

## Daily commands

| Goal | Command (repo root unless noted) |
|------|----------------------------------|
| Start / stop DB | `npm run db:up` / `npm run db:down` |
| Apply new migrations | `npm run db:migrate` |
| Wipe and rebuild the local DB | `cd db && npm run db:reset` (refuses non-localhost URLs) |
| See what's in the DB | `npm run db:check` |
| SQL rule tests | `npm run db:test` |
| Dev server | `npm run dev` |
| Everything CI checks | `npm run check` (lint + types + tests) and `npm run build` |

## Two databases on your machine?

You may already have PostgreSQL installed natively (pgAdmin, port 5432, configured in `db/.env`).
That is fine – the Docker database uses **5433**, so they never collide. Pick one for the web app by
setting `DATABASE_URL` in `web/.env.local`. We recommend the Docker one for the app and tests because everyone gets the same PostGIS version, and CI uses the same image.

## Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| `failed to connect to the docker API … dockerDesktopLinuxEngine` | Docker Desktop is not running. Start it, wait for the whale icon to settle, retry. |
| `/api/ready` returns 503 | DB not reachable: `docker ps` should show `iceberg-db` healthy; check `DATABASE_URL` port is 5433. |
| `Invalid environment configuration: DATABASE_URL …` | `web/.env.local` missing or has a typo (the message lists each bad variable). |
| `PostGIS is not installed` from db scripts | You are pointing at a native Postgres without PostGIS. Use the Docker URL, or install PostGIS via Stack Builder. |
| `password authentication failed` | Special characters in the password must be URL-encoded (`@` → `%40`). |
| Port 3000 busy | `npm run dev -- -p 3001`. |
| Changes to `.env.local` not picked up | Restart `npm run dev`. |

## Where to look when something is confusing

1. `docs/02-architecture.md` – where does this code belong?
2. `docs/05-backend-guide.md` – how do I add an endpoint?
3. `web/node_modules/next/dist/docs/` – the Next.js docs that match the *installed* version.
