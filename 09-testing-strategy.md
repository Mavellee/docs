# 09 – Testing strategy

Goal: be able to change code on Friday and ship on Monday without fear. We test **behaviour that
matters**, at the cheapest level that can catch it.

## The pyramid

```
            ▲  few, slow, high confidence
           ╱ ╲   E2E (Playwright)         ~10-20 journeys, run against staging
          ╱───╲
         ╱     ╲  Integration              real Postgres: repositories, SQL functions, RLS, API routes
        ╱───────╲
       ╱         ╲ Unit (Vitest)           services, Zod schemas, pure helpers, apiRoute wrapper
      ╱───────────╲   many, milliseconds
```

| Level | Tool | Runs | Real DB? | Example |
|-------|------|------|----------|---------|
| Unit | Vitest | every commit, CI, locally on save | no | `apiRoute` turns `ConflictError` into 409; Zod rejects a 17-year-old |
| SQL tests | `db/tests/smoke_test.sql` via `npm run db:test` (pgTAP later) | CI `db` job | yes (throw-away container) | already covers review vs instant approval, discover filters, needs-info → resubmit, block, under-18, audit append-only, account purge. **Gap to add in M3:** two *concurrent* `like_user` calls (needs two connections, so a Vitest integration test, not this single-transaction script) |
| Integration | Vitest with `TEST_DATABASE_URL` (`*.int.test.ts`) | CI `web` job | yes | repository queries match the schema; discover excludes blocked users |
| RLS tests | SQL tests, impersonating roles | CI `db` job (from M1) | yes | user A cannot read user B's messages |
| E2E | Playwright | staging after each deploy, nightly | staging DB | sign up → onboarding → like → match → chat |
| Manual | checklist in doc 04 | before promoting to production | staging | exploratory + mobile devices |

## What must be tested (by area)

| Area | Must-have tests |
|------|-----------------|
| Auth / session | no session → 401; suspended user → 403; role mismatch → 403; user id never taken from body |
| Validation | every Zod schema: valid, boundary, invalid; DOB exactly 18 today / yesterday |
| Matching | one-sided like → no match; mutual → exactly one match; concurrent mutual likes; blocked pair; unmatched pair never resurrects; review vs instant approval |
| Astrologer | two astrologers claim at once → distinct reviews; cap on open reviews; wrong astrologer cannot submit; needs_info → resubmit creates attempt 2 |
| Chat | non-participant cannot read/send; closed conversation; idempotent send (same client UUID twice → one row); read pointer moves forward only |
| Events | capacity boundary (N and N+1 registrations at the same time), waitlist promotion order, QR single use |
| Privacy | astrologer view exposes no phone/email; deleted user disappears from Discover immediately; purge cascades; sun sign hidden unless allowed |
| API contract | error body shape and `x-request-id` on every failure path |
| Moderation / AI | flagged photo stays hidden; LLM failure falls back to library; budget cap respected (with fakes) |

## Test data and fakes

- **Fixed dev seed** (`db/seed/dev_data.sql`) gives stable people (Aanya, Arjun, …) with known UUIDs – use them in integration tests; tests that mutate data run inside a transaction that is rolled back, or create their own users.
- **Fake integrations:** each module in `src/server/integrations/*` exports an interface and a fake. Tests inject the fake; local dev uses the fake by default. Real clients are exercised only in staging.
- **No snapshots of big HTML.** Assert on behaviour and roles (`getByRole`), not CSS classes.
- **Time:** inject a clock for anything time-based (OTP expiry, review due dates, prompt scheduling).

## Writing a good test (checklist)

1. Name states the rule: `"does not create a match when one user has blocked the other"`.
2. Arrange / Act / Assert, one behaviour per test.
3. Test the unhappy path as carefully as the happy one – most real bugs live there.
4. No sleeping; use deterministic waits.
5. A bug fix PR starts with a failing test that reproduces it.

## Commands

```bash
npm test                      # from repo root: web unit tests
cd web && npm run test:watch  # watch mode while coding
cd web && npm run check       # lint + typecheck + tests (run before pushing)
npm run db:test               # SQL smoke tests (needs the local DB: npm run db:up && npm run db:setup)

# integration tests against your local Docker database:
#   macOS/Linux/Git Bash:   TEST_DATABASE_URL=postgresql://postgres:postgres@localhost:5433/iceberg npm test
#   PowerShell:             $env:TEST_DATABASE_URL="postgresql://postgres:postgres@localhost:5433/iceberg"; npm test
```

## Quality gates

| Gate | Where | Blocks |
|------|-------|--------|
| ESLint, TypeScript, unit + integration tests, build | CI on every PR | merge into `staging` |
| DB migrations + SQL tests on empty database | CI | merge |
| Migration applied to staging before the app deploy | `migrate.yml` | promotion |
| Playwright smoke on staging (sign-in → discover → settings) | after staging deploy (M3+) | promotion to `main` |
| Manual release checklist (doc 04) | promotion PR | production deploy |
| Coverage | informational at first; target ≥ 80 % on `src/server/**` | – |

## Performance and load (before launch, M8)

- k6 script against staging: 200 concurrent users browsing Discover, 50 chats exchanging messages. Targets: p95 API < 300 ms, error rate < 0.5 %, DB CPU < 60 %.
- `EXPLAIN ANALYZE` on `discover_profiles` with 100 k seeded profiles; add/adjust indexes from the results.
- Verify connection counts stay under the pooler limit while scaling Vercel instances.
