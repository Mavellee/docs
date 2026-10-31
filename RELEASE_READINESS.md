# Mavelle — release readiness and QA sprint 1

Date: 30 October 2026 · Branch: `feature/knot-platform` (base commit `74834f3`) · Environment: local development (Docker PostgreSQL 16, API :4000, web :5180). No production environment exists yet; nothing was deployed.

## 1. Readiness summary

**Recommendation: needs fixes before beta, then ready for a limited private beta.** The code is in good shape. A 67-step two-user journey passes end to end, and so do 133 server tests and 58 web tests. Every security property claimed below is backed by a test.

What blocks inviting real people is mostly setup, not code:

1. An email provider must be configured. Without it, nobody can verify their account.
2. The support addresses are placeholders.
3. The legal pages are unreviewed drafts.

On top of that, video and touch reordering still need checking on real phones (section 9). Once the items in section 8 are done, Mavelle is suitable for a **limited, invite-only private beta** with the limits in section 5.

Under the QA Sprint classification: **Ready for further manual verification**.

## 2. Baseline (recorded before any change)

| Check | Command | Result |
|---|---|---|
| Git | `git status` | Clean; branch level with origin at `74834f3` |
| Secrets in git | `git ls-files` for env files | Only `*.env.example` files are tracked; `.env` is ignored |
| Dependencies | `npm ls --workspaces --depth=0` | All present and valid |
| Vulnerabilities | `npm audit --omit=dev` | **5 high**: nodemailer, zod, prisma → @prisma/config → deepmerge-ts |
| Prisma | `prisma validate`, `migrate status`, `migrate diff` (migrations vs schema) | Valid; 7 migrations applied; no drift |
| Type check | `npm run typecheck` | Pass |
| Lint | `npm run lint` | Pass |
| Server tests | `npm test --workspace server` | **127 / 128**: one failure in `redflag.acceptance` test 5 (timestamp ordering, see F5) |
| Web tests | `npm test --workspace web` | 58 / 58 |
| Production build | `npm run build` | Pass |

Architecture, as found:
- npm workspaces: `server` (Express 5, TypeScript, Prisma 6, Socket.IO 4, vitest + supertest) and `web` (React 19, Vite 7, Tailwind 3, vitest + Testing Library + axe).
- Sessions are opaque HttpOnly cookies. Media lives on a disk volume, with only keys stored in PostgreSQL.
- Production runs Docker Compose (Caddy, web, api, postgres) on one Lightsail instance.

## 3. Findings

Severity: **P0** critical security or data loss · **P1** release-blocking · **P2** important · **P3** polish.

### Fixed in this sprint

| # | Issue | Sev | Evidence | Fix | Test | Result | Remaining risk |
|---|---|---|---|---|---|---|---|
| F1 | Backups didn't include media. `backup.sh` dumped only PostgreSQL, so a restore would leave every profile pointing at missing photos and videos. | P1 (data loss on recovery) | `deploy/backup.sh` ran `pg_dump` only; the `uploads` volume was never archived | `backup.sh` archives the uploads volume with the same timestamp as each dump, and rotates both. `restore.sh` restores the matching archive, and warns if it's missing. Docs updated. | Archive and extract round trip run with the real `knot-api` image as its non-root user, against throwaway volumes; `bash -n` syntax check on both scripts | Pass | A full restore hasn't been rehearsed on a scratch instance. Backups still sit on the same host until copied off. |
| F2 | Without SMTP configured, production would **log every verification and password-reset link** (live account tokens) and keep every email in memory forever. | P1 (security) | `ConsoleMailer` logged `mail.text` in every non-test mode and pushed to an unbounded `outbox` | In production it logs only "Email NOT sent" (no body, no address) and keeps nothing. It also logs a loud startup error when `SMTP_HOST` is missing. Development behaviour is unchanged. | `tests/mailer.test.ts` (2 tests) | Pass | Emails still aren't **sent** until SMTP is configured (blocker B1). |
| F3 | nodemailer 6.10.1 had about 20 advisories, including SMTP command injection, recipient-domain bypass and credential disclosure. | P1 (security) | `npm audit --omit=dev` | Upgraded to nodemailer **10.3.1**. Our code uses only `createTransport` and `sendMail`. | Type check; a v10 `sendMail` through an in-memory transport using the same options shape; `npm audit` | Pass: the nodemailer advisories are gone | Real SMTP delivery is untested (no provider yet). |
| F4 | An account with **no profile** could like or pass on members through the API. | P2 | Two-user run: liking from an unpublished account returned 200 | Likes and passes now require the actor to have a published profile. | `matching.test.ts` "requires the liker to have a published profile"; journey step 7 | Pass (400 with a clear message) | None known |
| F5 | A game's reveal time could be recorded up to 1 ms **before** the answer that triggered it (two clocks), making a test flaky. | P2 | Baseline failure in `redflag.acceptance` test 5 | Submission and reveal now use one timestamp. | That test, run 5 times in a row, then the full suite | Pass | None |
| F6 | Uploads had no limit of their own: one member could replace a 50 MB video repeatedly, bounded only by the 300-requests-a-minute API limit. | P2 | `media.ts` routes used only the general API limiter | New per-member `uploadLimiter` (30 per 10 minutes), checked before any file data is read | `media.test.ts` "rate-limits uploads per member" | Pass | Limits are in memory, so a single API instance only (see O6). |
| F7 | Login limits were per IP only (20 per 15 minutes across login, signup and verification), so people on one network locked each other out. It happened twice during this QA run. | P2 | Two journey runs were blocked by 429s | Network limit raised to 100 per 15 minutes. Login and password reset now also limited per account and IP (10 per 15 minutes), so guessing one account stays slow. | `auth.test.ts`: the existing lock-out test, plus a new test that a different account on the same IP can still log in | Pass | Distributed guessing across many IPs is only slowed by Argon2 cost (no CAPTCHA). |

### Open — must be resolved before inviting members (deployment blockers)

| # | Issue | Sev | Evidence | Recommended action |
|---|---|---|---|---|
| B1 | No email provider is configured, so verification and reset emails won't be sent and new members can't verify. | P1 | `deploy/.env.production.example` has an empty `SMTP_HOST`; startup now logs an error (F2) | Choose a provider (some are paid, so this needs your approval), set the `SMTP_*` values, and send a test verification on staging. |
| B2 | Support, safety and privacy addresses are placeholders (`@knot.example`). | P1 | `web/src/pages/public/ContentPages.tsx` | Replace them with real monitored inboxes. |
| B3 | Terms and Privacy pages are drafts. | P1 | Labelled as drafts in the app | Legal review before inviting anyone. |
| B4 | No restore has ever been rehearsed. | P1 | `docs/DEPLOYMENT.md` asks for one | Restore a database and media pair on a scratch instance before launch. |

### Open — important, not blocking a small invite-only beta

| # | Issue | Sev | Evidence | Recommendation |
|---|---|---|---|---|
| O1 | After an unmatch, either person can still open the other's profile and media from a saved link. Discover and messaging are correctly closed. | P2 | Earlier audit, still true (`GET /profiles/:id` checks blocks only) | Treat an ended match like a block for profile and media views. This is a product decision, previously deferred. |
| O2 | No pre-moderation: photos **and videos** are visible as soon as they're uploaded; moderators can only reject afterwards. | P2 | `ProfilePhoto.status` defaults to `APPROVED` | For a beta, a moderator "recent uploads" review list at minimum. Videos raise the stakes. |
| O3 | Photos are served at full upload size (up to 5 MB) everywhere, including 40px avatars and Discover. | P2 | `/api/photos/:id` streams the original | Add a resizing step (thumbnail and display sizes) at upload. |
| O4 | zod 3 advisory: memory exhaustion validating oversized constrained collections. | P2 | `npm audit` | Partly mitigated by the 100 KB JSON body limit. Plan the zod 4 migration (breaking). |
| O5 | Prisma tooling advisory (deepmerge-ts inside `@prisma/config`). | P3 | `npm audit` | Affects only the CLI and config loading, not user input. Upgrade with the Prisma 7 migration. |
| O6 | Single-instance design: rate limits, presence and Socket.IO rooms are in memory. | P2 | `docs/DEPLOYMENT.md` | Keep one API container for the beta. Add Redis before scaling. |
| O7 | Purging deleted accounts after 30 days isn't scheduled. | P2 | Documented as a manual or cron step | Add the cron job next to the backup job. |
| O8 | Videos are stored as uploaded (no transcoding). iPhone `.mov` files are refused, as are WebM files with no recorded length. | P2 | `server/src/lib/video.ts`, `docs/ARCHITECTURE.md` | Acceptable for a beta. Add ffmpeg processing later. |
| O9 | One realtime delivery miss in the first journey run that couldn't be reproduced. | P2 | Passed in every later attempt: 3 direct trials, a new-match repro and 2 full journeys | Watch client "reconnecting" states during the beta. Socket.IO reconnects and REST refetches recover messages (verified). |
| O10 | Files from a crashed upload can stay in `UPLOAD_DIR/.incoming`. | P3 | They're cleaned on every request path, but not if the process dies mid-upload | Sweep `.incoming` at startup. |
| O11 | Date planning isn't implemented; there's no verified badge because no verification exists. | P3 | No routes, models or UI | Product backlog. |

**No P0 issue was found.** Object-level authorization, hidden game answers, media visibility, session revocation and upload validation are all covered by tests (section 4).

## 4. What works, and the evidence

The **journey** is a scripted run against the live API with two brand-new accounts, a third profile-less account, a seeded outsider (Aanya) and seeded expert and admin accounts. Final run: **67 / 67 passed**. **Tests** refers to the automated suites.

| Feature | Evidence |
|---|---|
| Signup, verification, login, logout, session persistence | Journey §1; `auth.test.ts`. Logout invalidates the session; wrong password → 401; deleted accounts → 403 "scheduled for deletion" |
| Session revocation also cuts live sockets | `chat.test.ts` (logout, revoked session, suspension) |
| Profile creation and editing, with validation | Journey §2. Bio over 500 characters → 422; publishing without a photo → 400; edits persist |
| Multiple photos: order, captions, deletion | Journey §3; `media.test.ts`. Captions stay on the right photo after reordering; deleting compacts positions and removes the file from disk; others can't edit or delete (404) |
| Main photo chosen separately | Journey §3 and §5 (`photoIds[0]` is the chosen photo); `media.test.ts` (videos can never be main; deleting the main photo promotes the next) |
| Video upload, poster, playback, deletion | Journey §4: a 6-second MP4 accepted; a 35-second video and a fake video rejected; playback streams with Range requests (206). `media.test.ts`: WebM accepted, MP4 location metadata removed, replace keeps the caption, delete removes the video and poster files. Browser (previous session): poster captured client-side, no video bytes fetched until play |
| Full-profile Discover | Journey §5: no duplicates across pages; media in the owner's order including the video and its caption; date of birth and email absent. `media.test.tsx`: one person at a time, Pass advances, empty state |
| Compatibility from real data | Journey §6: score computed only from answers both people gave, the same across requests. `compatibility.test.ts` (engine). UI: "Still discovering" when there's no score (`story.test.tsx`) |
| Likes, passes, duplicates, matching | Journey §7: a duplicate like is idempotent; two simultaneous like-backs create **one** match row; no conversation before a match |
| Likes on a specific photo, video or prompt | Journey §7: a like on a video with a note shows to the recipient, then becomes the first message with the media reference. `story.test.ts`, `media.test.ts` |
| Messaging is locked to matched members | Journey §8: an outsider can't read, post or join the socket room; no conversation exists before a match |
| Realtime chat, refresh, reconnect | Journey §8: live delivery; a message sent while offline appears after reconnect and refetch. `chat.test.ts` |
| In-chat games | Journey §9: the partner's answer is `null` until both answer; duplicate answers → 409; revealed together with a discussion prompt; outsiders get 404. `redflag.acceptance.test.ts` (16), `chatgames.test.ts` (16) |
| Blocking and reporting | Journey §10: report → 201; a block closes chat both ways, hides profiles and media both ways, and cancels the live game. `safety.test.ts` |
| Account deletion | Journey §11: login refused, media hidden, removed from Discover; files kept 30 days then purged (`safety.test.ts`, `purgeDeleted.ts`) |
| Media privacy | `media.test.ts`: rejected items, unpublished profiles and blocked members get 404; owners and moderators still see them |
| Responsive layout and accessibility | Sweep of 11 screens at 375 / 768 / 1440 px: no sideways overflow, no broken images, no unlabelled controls, no error banners. axe checks in web tests; colour contrast test (23 pairs) |
| Migrations | All 7 apply; no drift between migrations and schema; the last migration is additive with a non-destructive backfill |

## 5. Known limitations for a beta

- One API instance only (O6).
- No media pre-moderation (O2).
- Full-size images (O3).
- Unmatched profiles are reachable by direct link (O1).
- No transcoding (O8).
- Date planning isn't available (O11).

## 6. Files changed in this sprint

- `deploy/backup.sh`, `deploy/restore.sh`, `docs/DEPLOYMENT.md` — paired database and media backups (F1).
- `server/src/lib/mailer.ts` — no token logging or unbounded outbox in production; startup error (F2).
- `server/package.json`, `package-lock.json` — nodemailer 10.3.1 (F3).
- `server/src/modules/matches/service.ts` — only published members can like or pass (F4).
- `server/src/modules/chatgames/service.ts` — one timestamp for submission and reveal (F5).
- `server/src/middleware/security.ts`, `server/src/modules/profiles/media.ts`, `server/src/modules/auth/routes.ts` — upload limiter; per-network and per-account auth limits (F6, F7).
- Tests: `server/tests/mailer.test.ts` (new), `auth.test.ts`, `matching.test.ts`, `media.test.ts`.
- `docs/RELEASE_READINESS.md` (this report).

**Database migrations created: none.** No data was reset or deleted. Only throwaway Docker volumes created for the backup test were removed afterwards.

## 7. Commands executed

- `npm ls`, `npm audit --omit=dev`
- `prisma validate`, `prisma migrate status`, `prisma migrate diff --from-migrations … --shadow-database-url …/knot_shadow`
- `npm run typecheck`, `npm run lint`, `npm test --workspace server|web`, `npm run build`
- `npm install nodemailer@^10.3.1 --workspace server`
- `bash -n` on both deploy scripts, and an archive/extract round trip through `docker run knot-api:latest` with throwaway volumes
- The two-user journey script (scratch, not committed) against the live dev API, run five times:
  1. Stopped on a syntax error in my script.
  2. 64 of 67: failed on F4, on one realtime miss (O9) and on one wrong test expectation (deleted accounts correctly get 403, not 401).
  3. Blocked by the per-IP login limit (F7).
  4. 66 of 67: failed only on F4.
  5. After the fixes: **67 / 67**.
- A browser sweep of 11 screens at 3 widths
- The dev API was restarted once to clear in-memory rate limits

**Final results:** type check ✔ · lint ✔ · server **133 / 133** ✔ · web **58 / 58** ✔ · production build ✔ · Prisma valid, no drift ✔ · journey **67 / 67** ✔ · `npm audit --omit=dev`: 4 high remaining (zod and the Prisma toolchain; see O4 and O5).

## 8. Before inviting beta members, in priority order

1. Configure SMTP and send a real verification email on staging (B1).
2. Replace the placeholder support, safety and privacy addresses (B2).
3. Legal review of the Terms and Privacy pages (B3).
4. Deploy to a staging Lightsail instance. Rehearse backup → restore with a database and media pair, and schedule the backup, off-site copy and purge cron jobs (B4, O7).
5. Manual tests on real devices (section 9).
6. Decide on unmatched-profile visibility (O1) and add a moderator review list for new uploads (O2).
7. Next engineering work: image resizing (O3), zod 4 and Prisma 7 upgrades (O4, O5), sweeping `.incoming` at startup (O10).

## 9. Manual tests still needed

- iPhone (Safari) and Android (Chrome):
  - Upload a phone-recorded video.
  - Confirm the poster appears.
  - Play with sound, mute and fullscreen.
  - Confirm nothing plays until tapped.
- Touch drag-to-reorder in the media workspace (arrow buttons are tested).
- Real SMTP delivery: verification and reset emails reach inboxes and aren't marked as spam.
- A full restore on a scratch instance.
- Two people on two physical devices: match, chat and play a game over mobile data, including a network drop mid-game.
- Screen-reader pass (VoiceOver / TalkBack) through onboarding, Discover and chat.

---

# QA sprint 2 (30 October 2026)

**Starting point:** commit `0d3120d`, a clean working tree. All checks passed as committed:
- type check, lint and production build;
- server tests 133/133 and web tests 58/58;
- Prisma valid, with no drift between migrations and schema.

## New probes

| Probe | How | Result |
|---|---|---|
| Every API endpoint without a session | 106 endpoints found in the router source and called anonymously | All non-public endpoints → 401 |
| Malformed IDs and empty bodies | Each endpoint with an `:id` called with `not-a-uuid`; every write called with `{}` | **No 500s** |
| Unknown or foreign IDs | Each `:id` endpoint called with a random UUID | 4 return 200, all owner-scoped idempotent writes (`deleteMany` / `updateMany` filtered by the caller's id): remove a compatibility answer, mark a notification read, unblock, revoke a session. Not an access-control hole. |
| Expired sessions | Session moved into the past | API → 401; socket refused (`hardening.test.ts`) |
| Password change | Two devices, change on one | The other is signed out; the current one stays (`hardening.test.ts`) |
| Everything a blocked person might try | Like, view profile and media, message (REST and socket), open the game, join the room | All refused; their earlier like disappears from Likes; reporting stays possible (`hardening.test.ts`) |
| Interrupted upload | Connection dropped mid-way through a video upload | No partial file left in `.incoming`; no database row (`hardening.test.ts`) |
| Rapid messages | 12 sends from one device without waiting; 35 concurrent sends from two people | **Failed before the fix**; see S1 |
| Chat after reconnect | API stopped; a message written straight to the database (so no live event exists); API restarted | **Message never appeared before the fix**; see S2 |

## Fixes

| # | Issue | Sev | Evidence | Fix | Test | Result |
|---|---|---|---|---|---|---|
| S1 | Messages one person sent quickly were **stored out of order** (m0, m3, m1, m5…). Messages created in the same millisecond also had no stable order, so "load older" paging could skip a message at the page boundary. | P2 | `hardening.test.ts`: stored order `m0,m3,m1,m5,…` | Each socket handles its own sends one at a time, in arrival order. Every message (chat, game card, first message on a match) takes a per-conversation lock and a timestamp strictly after the previous one (`nextMessageTime`), with no schema change. | 12 rapid sends stay in order; 35 concurrent sends from two people get 35 distinct timestamps, each person's order preserved, paging 30 + 5 returns all 35 exactly once, both people see the same sequence. Run 3 times. | Pass |
| S2 | After a dropped connection, the chat rejoined the room but **never fetched messages sent while offline**; they appeared only after a full page refresh. | P2 | Manual check in the browser with the API stopped and restarted | On every reconnect the chat refetches the latest page and merges it with what's on screen (`mergeMessages`: no duplicates, pending messages replaced by the server copy, unsent ones kept at the end). Live events use the same merge. | `web/src/test/messages.test.ts` (3); live browser check: after the restart the message appeared once and the "Reconnecting" banner cleared, without a page refresh | Pass |

**Files changed:**
- Server: `server/src/modules/chat/service.ts`, `server/src/modules/chatgames/service.ts`, `server/src/modules/matches/service.ts`, `server/src/realtime/socket.ts`
- Web: `web/src/lib/messages.ts` (new), `web/src/pages/app/Messages.tsx`
- Tests: `server/tests/hardening.test.ts` (new), `web/src/test/messages.test.ts` (new)

**Database:** no schema change and no migration. The per-conversation lock is a transaction-scoped PostgreSQL advisory lock.

## Results after the fixes

| Check | Result |
|---|---|
| Type check / lint | Pass / pass |
| Server tests | **139 / 139** (6 new) |
| Web tests | **61 / 61** (3 new) |
| Production build | Pass |
| Prisma | Valid, no drift, migrations up to date |
| `npm audit --omit=dev` | 4 high, unchanged (zod; Prisma toolchain); see O4, O5 |

**Not run:**
- No browser end-to-end suite exists in the repository.
- No real SMTP send (no provider is configured).
- No real phones.

## Recommendation (unchanged)

**Needs further verification:**
- Blockers B1–B4 above.
- The manual tests in section 9.

After those, Mavelle is suitable for a **limited private beta**.

O9 (one unexplained realtime miss in sprint 1) is now much less likely to matter for members, because S2 restores missed messages on reconnect.
