# 08 – Security and privacy

A dating app holds the most sensitive kind of data (identity, photos, location, orientation signals,
birth details, private conversations) about people who are often young. Security is a feature, not a phase.

## Non-negotiable rules

1. **18+ only.** Enforced in the database (`enforce_min_age` trigger), the API schema and the UI. Reports with reason `underage` auto-suspend the account for review.
2. **Identity comes from the session, never from the request.** No endpoint accepts "which user am I".
3. **Private by default.** Photos, voice intros and chat media live in *private* buckets and are served by signed URLs that expire in minutes. Nothing sensitive is ever at a guessable public URL.
4. **Exact location is never stored.** Store a *rounded* point (about 1 km grid); show only "3 km away".
5. **Astrology data exists only with consent**, can be withdrawn (row deleted), and only the assigned astrologer sees it through `astrologer_review_view`.
6. **Secrets stay on the server.** Only `NEXT_PUBLIC_*` reaches the browser; reviewed in every PR that touches `env.public.ts`.
7. **Least privilege.** The app's DB role cannot drop tables; the service-role key is used only in named admin/system modules.
8. **Every sensitive action is auditable** (`audit_log`, append-only by trigger).
9. **Never log** OTPs, tokens, full phone/email, message bodies, birth data (pino `redact` is a backstop).
10. **Fail closed.** Unknown role, missing session, unparseable input → deny.

## Threat model (what we defend against)

| Threat | Example | Defences |
|--------|---------|----------|
| Account takeover | SIM-swap / OTP brute-force | OTP attempt limits (`auth_otps.attempts` / Supabase), rate limit per phone + IP, short OTP TTL, session revocation, notify on new device |
| Enumeration / scraping | Bot pulls every profile | Rate limits on `/discover`, only candidates matching *both* users' preferences are returned, no public profile URLs, Cloudflare bot protection + Turnstile at sign-up |
| IDOR (insecure direct object reference) | Change `:id` to read someone else's match/chat | Ownership check in every service (participant / party-to-match), RLS as second wall, 404 not 403 for things you may not know exist |
| Injection | SQL in a bio, XSS in a message | Parameterised SQL only; React escapes output; no `dangerouslySetInnerHTML`; CSP (M1); length limits via Zod and DB `check`s |
| Privilege escalation | User calls astrologer endpoint | Declarative `auth: ["astrologer"]`; role stored server-side (`users.role`), never trusted from the client; role changes only by admin and audited |
| Harassment / abuse | Spam messages, unsolicited explicit images | Mutual-match-only chat, block/report on every message, image + text moderation, rate limits, auto-lock of flagged conversations |
| Fake / underage profiles | Catfishing | Phone verification, optional college-email verification, photo moderation, verified badge, report flow |
| Data leak via storage | Guessable photo URL | Private buckets, per-object signed URLs, storage policies tied to ownership/match |
| Abuse of paid APIs | Loop calling the LLM or SMS | Per-user + global budgets, rate limits, circuit breakers |
| Supply-chain | Malicious npm package | Lockfiles committed, `npm ci` in CI, Dependabot + `npm audit` in CI (M1), pin GitHub Actions by major version, review new dependencies |
| Insider access | Staff browsing profiles | Astrologers see only the review view; moderators see only reported content; admin reads are logged |

## Authorisation matrix (who may read what)

| Data | Self | Matched user | Other users | Astrologer (assigned) | Moderator | Admin |
|------|------|--------------|-------------|-----------------------|-----------|-------|
| Profile (name, bio, photos) | ✅ | ✅ | ✅ only if mutually discoverable | name, age, gender only | ✅ if reported | ✅ |
| Phone / email | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ (audited) |
| Exact birth details | ✅ | ❌ | ❌ | ✅ via view | ❌ | ✅ (audited) |
| Sun sign | ✅ | if `show_astrology_details` | if `show_astrology_details` | ✅ | ❌ | ✅ |
| Messages | ✅ own conversations | ✅ | ❌ | ❌ | only reported messages | ❌ by default |
| Review internal notes | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ |
| Reports about me | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |

## Row Level Security (Milestone 1 migration `011_rls.sql`)

RLS protects the paths where the browser talks to Supabase directly (Realtime, Storage) and is a safety net for the API.
Principles:
- `alter table ... enable row level security` on **every** table; no policy = no access.
- The API's database role is a dedicated role with exactly the grants it needs; Supabase's `service_role` is only used by background jobs.
- Policies use `auth.uid()` and helper functions such as `is_participant(conversation_id)` (security definer, `search_path` fixed).
- **Every policy has a pgTAP/SQL test** in `db/tests/` that proves both "allowed" and "denied" (doc 09). Example tests: user A cannot `select` user B's messages; a blocked user cannot subscribe to the conversation.

## Data protection (India DPDP Act-style practices)

| Requirement | How we meet it |
|-------------|----------------|
| Consent, specific and withdrawable | `consents` table: one row per purpose (`terms`, `privacy`, `astrology_processing`, `ai_prompts`, `marketing`) with policy version; withdrawing = new row `granted=false` which immediately disables the feature and deletes the data |
| Purpose limitation | Astrology data used only for review; chat content used for AI prompts only with `ai_prompts` consent, and only the minimum context |
| Right to access / portability | `POST /me/export` → ZIP of the user's data (profile, matches, messages they sent) delivered by signed link, expires in 24 h |
| Right to erasure | `request_account_deletion()` → 30-day grace → `purge_deleted_accounts()` + storage purge; backups age out within the retention window (documented in the privacy policy) |
| Data minimisation | No exact location, no unnecessary fields, EXIF stripped from photos |
| Children | 18+ gate, underage report path |
| Breach readiness | Incident runbook (below); Sentry/alerting; audit log; ability to revoke all sessions |
| Data residency | Supabase region Mumbai if available; otherwise Singapore (state this in the privacy policy) |
| Vendors | List sub-processors (Supabase, Vercel, Twilio/MSG91, Resend, LLM vendor, Sentry, PostHog) in the privacy policy; sign DPAs where offered |

Legal copy (Terms, Privacy Policy, Community Guidelines) must be reviewed by a lawyer before launch; the technical
design above is built so the policies can truthfully say what we do.

## Sending data to an LLM

- Send the **minimum**: prompt level, tags, shared-interest names. Never names, phone, photos, birth data, or raw chat history beyond the last few anonymised turns, and only with `ai_prompts` consent.
- System prompt forbids sexual, medical, legal, self-harm and politically sensitive questions; output passes a moderation check; fallback to the library.
- Log model name and prompt id (`conversation_prompts.model`), not the full prompt text.

## Web security checklist (applied in M1)

- [ ] Content-Security-Policy with nonces (Supabase, PostHog, Sentry hosts allow-listed)
- [x] `X-Frame-Options: DENY`, `nosniff`, `Referrer-Policy`, `Permissions-Policy`, HSTS (in `next.config.ts`)
- [ ] Cookies: `httpOnly`, `secure`, `sameSite=lax` (Supabase SSR defaults – verify)
- [ ] CSRF: state-changing routes accept only `application/json` and check `Origin` against `NEXT_PUBLIC_APP_URL`
- [ ] CORS: none needed (same origin). Do not add `Access-Control-Allow-Origin: *`.
- [ ] Rate limiting on auth, likes, messages, reports, uploads
- [ ] Cloudflare Turnstile on sign-up and OTP request
- [ ] Dependabot + `npm audit --omit=dev` in CI
- [ ] Verify database TLS with `DB_SSL_CA` (currently `rejectUnauthorized:false` if no CA is supplied – acceptable for first staging, **not** for production launch)

## Incident runbook (one page, fill in names before launch)

1. **Detect** – Sentry alert / uptime alert / user report. 2. **Contain** – disable the feature flag (PostHog) or revoke sessions / rotate the key. 3. **Assess** – use `audit_log` and request-id logs to scope who/what. 4. **Notify** – affected users and the regulator within the legally required time. 5. **Fix** – hotfix branch from `main`. 6. **Review** – write a blameless post-mortem in `docs/incidents/`.
