# Mavelle private-beta functional audit

Date: 29 October 2026 · Branch: `feature/knot-platform` · Environment: local dev (Docker Postgres, API on :4000, web on :5180)

## How this was verified

Nothing below is marked as working unless it was exercised. Evidence comes from four sources:

| Source | What it covered |
|---|---|
| Automated checks | `npm run typecheck`, `npm run lint`, `npm test` (server 100, web 52), `npm run build` — all pass |
| Live API run | 80 scripted checks against the running API with two brand-new accounts plus seeded staff: registration → verification → profile → photos → publish → login persistence → discovery filters → likes → mutual match → chat (REST and Socket.IO) → in-chat game privacy → expert workflow → unmatch. Plus 23 checks on settings, notifications, compatibility, AI fallback, reports, blocks, staff permissions, password reset and data export |
| Route render sweep | Every frontend route loaded in the browser pane: no uncaught errors, no blank pages, staff routes redirect members |
| Code review | Route guards, socket authentication, upload handling, CSRF, rate limiting, mailer, placeholders |

Earlier sessions also covered responsive layouts at 375/390/768/1024/1440 and a page-wide contrast scan.

## Fixed during this audit

| # | Severity | Issue | Fix | Test |
|---|---|---|---|---|
| F1 | **High (security)** | Logging out, revoking a session in Settings, password reset/change, suspension, ban and account deletion revoked the session in the database, but any **open Socket.IO connection stayed live and kept receiving private messages and notifications** until the client next sent something. | Sockets now join a per-session room; every revocation path disconnects those sockets immediately (`realtime/bus.ts`, `auth/service.ts`, `auth/routes.ts`, `settings/routes.ts`). The web client re-checks the session when the server drops it, so a revoked device lands on the login page instead of a permanent "Reconnecting…" banner (`web/src/lib/socket.tsx`). | 3 new tests in `server/tests/chat.test.ts` (logout, revoked session keeps the current one, suspension). Also verified in the browser: revoking the pane's session from another login sent it to "Welcome back". |
| F2 | Low | A `POST` with a malformed `Referer` header (and no `Origin`) crashed the CSRF check with a 500. | The origin check treats an unparseable Referer as no origin and returns 403 (`middleware/security.ts`). | New test in `server/tests/auth.test.ts`. |

## Open issues, highest priority first

### P1 — must resolve before inviting beta users

| # | Area | Evidence | Problem | Recommended fix | Tests needed |
|---|---|---|---|---|---|
| P1-1 | Email delivery in production | `server/src/lib/mailer.ts:36` uses SMTP only when `SMTP_HOST` is set, otherwise the console mailer; `deploy/.env.production.example` ships `SMTP_HOST=` empty | A production deploy without SMTP **silently logs verification and reset emails instead of sending them**. New users can never verify, so they can't use Discover, Likes, Chat or Games. | Choose an email provider and set the SMTP values. Make the API refuse to start in production without `SMTP_HOST`. *(Needs your choice of provider; some are paid.)* | Config test: production + no SMTP host → startup error. A manual end-to-end send on staging. |
| P1-2 | Support contacts | `web/src/pages/public/ContentPages.tsx:266-268` lists `support@`, `safety@` and `privacy@mavelle.example` | Placeholder addresses: beta users have **no working way to reach support or report a safety concern off-platform**. | Replace them with real monitored addresses (needs your input). | None (content). |
| P1-3 | Legal pages | Terms and Privacy pages are drafts, labelled as such on the page and in the footer | Inviting real users usually requires reviewed terms and a privacy policy, especially for 18+ verification, photos and expert data sharing. | Legal review before the beta invitation. | None (content). |

### P2 — should fix during the beta

| # | Area | Evidence | Problem | Recommended fix | Tests needed |
|---|---|---|---|---|---|
| P2-1 | Privacy after unmatch | Live check: after B unmatched A, `GET /api/profiles/:B` returned **200** for A. `profiles/routes.ts:259` and the photo route (`:288`) hide only on **blocks**. Discovery does exclude the pair. | Someone you unmatched can still open your full profile and photos from a saved link or browser history. Messaging is correctly closed and re-liking does not rematch. | Treat an ended match (`UNMATCHED` or `BLOCKED`) like a block for profile, photo, compatibility and AI-icebreaker views. *(This changes product behaviour, so it was left for your decision.)* | Server tests: profile, photo and compatibility return 404 to both people after an unmatch; still visible to everyone else. |
| P2-2 | Photo moderation | `schema.prisma:316` defaults `ProfilePhoto.status` to `APPROVED`; moderators can reject afterwards (`POST /api/admin/photos/:id/reject`) | Uploads are public as soon as the profile is published. Moderation is reactive only, via reports. | For a small beta, add an admin "recent photos" review list, or start photos as `PENDING` with a moderator approval queue. | Upload creates a pending photo, a pending photo is hidden from others, and approval shows it. |
| P2-3 | Login rate limit | `middleware/security.ts`: `authLimiter` allows 20 requests per 15 minutes **per IP** across login and registration. The audit hit it from one machine. | Beta testers sharing an office or campus network could lock each other out of logging in for 15 minutes. | Key the limiter on IP **and** email (e.g. 10 attempts per account per 15 minutes) with a higher per-IP ceiling. | Rate-limit test per account; a different account from the same IP still works. |
| P2-4 | Single-instance state | Rate-limit counters, online presence (`realtime/presence.ts`) and Socket.IO rooms are in memory; `docs/DEPLOYMENT.md:139` already notes this | Correct for one API container. Running two would split chat delivery, presence and limits. | Keep the beta on one API instance. Add the Redis adapter and a shared limiter store before scaling. | Multi-instance smoke test once Redis is added. |
| P2-5 | Deleted-account purge | `docs/DEPLOYMENT.md:108-111` documents a command that must be run | If nobody schedules it, accounts past the 30-day grace period aren't purged as the privacy page promises. | Schedule it (cron or a systemd timer on the Lightsail host) and log the result. | Existing purge script; add a test for the grace-period cut-off. |

### P3 — low risk or missing coverage

| # | Area | Evidence | Problem | Recommended fix | Tests needed |
|---|---|---|---|---|---|
| P3-1 | Photo IDs | `GET /api/photos/:id` lets any active member fetch an approved photo of an active user, even if the profile is unpublished, given its UUID | UUIDs are unguessable and appear only in profiles the viewer could already see, so the exposure is small. | Also require the owner's profile to be published, unless the viewer has an active match with them. | Photo of an unpublished profile returns 404 to strangers. |
| P3-2 | Untested paths | No automated tests for: change-password, admin photo reject, chat image attachments, ban (only suspend is tested), expert *complete* happy path in the UI | Implemented in code but **not verified** in this audit | Add tests before the beta. | Listed in the gap column. |
| P3-3 | Per-account brute force | No per-account lockout or CAPTCHA (IP limiter only) | Slow distributed guessing is possible; Argon2id makes each guess expensive. | Covered by P2-3. Add an alert on repeated failures. | Same as P2-3. |

## Feature status

Status: **Verified** = exercised live and/or by automated tests in this audit. **Partial** = works, with the noted gap. **Static** = content only, by design.

| Feature / route | Status | Evidence | Notes |
|---|---|---|---|
| Public pages `/`, `/how-it-works`, `/intentions`, `/safety`, `/about`, `/guidelines` | Static, renders | Route sweep | No fake statistics, testimonials or counters found |
| `/terms`, `/privacy` | Static draft | Route sweep | P1-3 |
| `/contact` | Partial | Code | Placeholder addresses (P1-2); no form, by design |
| 404 page | Verified | Route sweep | |
| Registration `/signup` | Verified | Live: 201; under-18 → 422; duplicate → 409; cross-origin → 403 | Age is computed on the server |
| Email verification `/verify-email` | Verified (dev) | Live: token works once, then rejected; unverified users get 403 from Discover | Production delivery blocked by P1-1 |
| Login, logout, sessions | Verified | Live: wrong password → 401, logged-out cookie → 401, session list in Settings | Socket teardown fixed (F1) |
| Forgot / reset password | Verified (request), tested (reset) | Live: same response for unknown emails; `auth.test.ts` covers reset | |
| Onboarding `/onboarding` | Verified | Earlier browser run + live API: profile fields, prompts, publish blocked without a photo | |
| Photo upload, reorder, framing | Verified | Live: JPEG 201, fake image 400, reorder persists, EXIF stripped, served `image/jpeg` | |
| Profile editing and persistence `/app/profile` | Verified | Live: values, publish state and photo order survive logout/login; invalid age range → 422 | |
| Discover `/app/discover` + filters | Verified | Live: intention, interest and age filters narrow results; filters can't widen saved preferences | |
| Profile view `/app/people/:id` | Partial | Live: hides date of birth and email; blocked → 404; missing profile shows an empty state | P2-1 (still reachable after unmatch) |
| Likes `/app/likes` | Verified | Live: one-way like is not a match; recipient sees the note; liking yourself is rejected; repeat like is idempotent | |
| Mutual matching `/app/matches` | Verified | Live: match only on the second like; conversation created; the like's note becomes the first message | |
| Unmatch | Verified | Live: no messages over REST or socket afterwards; live game cancelled; pair not rediscovered; re-like doesn't rematch | P2-1 |
| Chat `/app/messages` (REST) | Verified | Live: persisted, outsiders get 404, anonymous gets 401 | |
| Chat (Socket.IO) | Verified | Live: unauthenticated and foreign-origin connections rejected; outsiders can't join or send; real-time delivery; idempotent resend | F1 fixed |
| Chat image attachments | Not verified | Code only | P3-2 |
| In-chat games | Verified | Live: one session shared by both, partner's answer `null` until both answer, then revealed to both; outsiders get 404; one live game per conversation (409); cancelled on unmatch | Plus 16 server tests |
| Solo games `/app/games`, `/app/games/:id` | Verified (earlier) | Earlier browser run, `games.test.ts` | |
| Compatibility `/app/compatibility` | Verified | Live: questions load, answer saves; engine covered by `compatibility.test.ts` | |
| Compatibility Experts (member) `/app/experts` | Verified | Live: request needs consent; another member can't cancel it; snapshot contains no messages or email | |
| Expert workspace `/app/expert-workspace` | Verified | Live: members and pending applicants get 403; queue visible to an approved expert; details only after claiming; claim works | Completion covered by `experts.test.ts` |
| Admin `/app/admin` | Verified | Live: members get 403, moderators see reports, admin overview loads; members are redirected away in the UI | Photo reject not tested (P3-2) |
| Safety: report, block, unblock | Verified | Live: report 201, block hides the profile (404), unblock works | |
| Settings `/app/settings` | Verified | Live: update persists, sessions list and revoke, data export 200; deletion covered by `safety.test.ts` | |
| Notifications `/app/notifications` | Verified | Live: loads after like/match | |
| Mavelle AI panels | Verified (fallback) | Live: with no provider, returns curated suggestions labelled "Suggestions from the Mavelle team"; `/ai/status` reports `available: false` | No paid AI dependency |

## Security review summary

- **Authentication:** Argon2id hashing; opaque HttpOnly session cookies (SameSite=Lax, Secure in production); sessions are hashed at rest and revocable; banned and pending-deletion users can't load a session.
- **CSRF:** non-GET requests must come from the app's own origin, using Origin or Referer (fixed F2).
- **Authorisation:** every router applies `requireAuth`, `requireActive` or `requireRole`. Conversation, game and expert access is re-checked per request. Outsiders get 404 rather than 403, so they can't learn that an ID exists.
- **Real time:** sockets are authenticated by the same cookie, origin-checked and re-validated on every action, and now disconnected on revocation (F1).
- **Uploads:** size and count limits, type detection from file content (not the extension), metadata stripped, random storage keys.
- **Data minimisation:** public profiles omit date of birth and email; expert snapshots contain only the fields the member consented to.
- **Open items:** P1-1, P2-1, P2-2, P2-3 and P3-1 above.

## Not covered by this audit

- A real production deploy, AWS, or delivery through a real email provider.
- Behaviour with more than one API instance.
- Real phones (layouts were checked in the browser's device emulation only).
- Drag-to-reorder photos in the browser (covered by a unit test and the API).
- Load and performance testing.

## Dev database side effects

The live run created these local dev records (nothing was deleted or reset):

- Accounts `audit-a-…@knot.test` and `audit-b-…@knot.test`, which have published profiles, an ended match, a cancelled game and a cancelled expert request.
- A report and a removed block by Aanya.
- A compatibility answer and settings toggles on audit-a.
- Tara's browser session was revoked during the socket test.
