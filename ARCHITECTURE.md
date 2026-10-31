# Mavelle — Architecture & Implementation Plan

## 1. What was in the repository

`main` contained a single, earlier concept called **Iceberg**: a raw-SQL PostgreSQL
setup in `db/` (10 hand-written migrations, PostGIS, astrologer review flow, campus
events, a `pg`-based migration runner, seeds and a SQL smoke test). There was no
backend server and no frontend.

**Decision:** `db/` is left untouched. It remains runnable exactly as before. Mavelle
uses its own Prisma schema and its own database (`knot`), so nothing in the Iceberg
database is migrated, altered or deleted. Reused ideas from Iceberg: ordered pair
rows for matches (`userAId < userBId`), append-only audit log, consent-gated
review, soft-delete with a purge window, rounded locations only.

Differences that made a straight reuse inappropriate:

| Iceberg (`db/`)                         | Mavelle                                                        |
|-----------------------------------------|-------------------------------------------------------------|
| Intent includes `open_to_astrology`     | Intentions are `SINGLE`, `SITUATIONSHIP`, `MARRIAGE`         |
| Astrologer review gates chat            | Expert review is optional and never gates matching or chat   |
| PostGIS required                        | Rounded lat/lng + bounding box + haversine (no extension)    |
| Phone/OTP auth, colleges, events        | Email + password (Argon2id), email verification              |

## 2. Layout

```
server/   Express + TypeScript + Prisma + Socket.IO (REST under /api)
web/      React + TypeScript + Vite + Tailwind + React Router
deploy/   Caddy reverse proxy, production compose, backup scripts
docs/     Architecture, API, environment, deployment
db/       Legacy Iceberg SQL (unchanged)
```

## 3. Key decisions

- **Sessions, not JWTs.** A random 32-byte token is set as an `HttpOnly`,
  `SameSite=Lax`, `Secure` (in production) cookie. Only its SHA-256 hash is stored
  in `Session`, so a database leak does not leak live sessions. Logout and password
  change revoke rows. Socket.IO authenticates with the same cookie.
- **CSRF.** State-changing requests must carry an `Origin` that matches
  `APP_ORIGIN` (plus SameSite=Lax cookies).
- **Passwords.** `argon2` (Argon2id, library defaults ≥ OWASP minimums).
- **Authorization lives in services.** Every conversation/message action checks
  that the caller is a member of an *active* match. IDs from the client are never
  trusted on their own.
- **Matching.** `Like` has a unique `(fromUserId, toUserId)`; `Match` has a unique
  ordered pair. The reciprocal check and match/conversation creation run in one
  Prisma transaction; concurrent duplicate inserts are caught via the unique
  constraint and resolved to the existing row.
- **Compatibility engine** is pure, deterministic and versioned
  (`server/src/modules/compatibility/engine.ts`). Weights come from config.
  Results (score, version, inputs, breakdown, timestamp) are stored in
  `CompatibilityResult` for audit. Missing answers are reported separately from
  mismatches. The score is never described as predicting relationship success.
- **Experts** are an account role (`EXPERT`) gated by admin approval of an
  `ExpertProfile`. Experts only see what a user explicitly chose to share on a
  request (a snapshot is stored on the request). They can never read chats.
- **AI** goes through `AIProvider` (`ollama`, `openai-compatible`, `none`). The
  product layer always has a curated fallback, enforces per-user and global daily
  limits, a concurrency cap, timeouts and output-length caps, and logs metadata
  only (`AIUsage` — no prompt or output text).
- **Profile media** (photos and short videos) share one table, `ProfilePhoto`, with
  a `kind`, an optional per-item `caption` (≤ 200 characters), a `position`, an
  `isPrimary` flag (images only, one per member, both enforced in SQL), and for
  videos a `durationMs` and `posterKey`. Files live on disk (volume) under random
  names; only keys and metadata are in PostgreSQL. Swappable for S3/Lightsail
  Object Storage.
  - Uploads stream to `UPLOAD_DIR/.incoming`, are validated **by content** (image
    magic bytes; MP4 `ftyp` brands or WebM EBML — never the extension or the
    client's MIME type) and removed from `.incoming` whatever the outcome.
  - Images have EXIF/XMP/IPTC stripped. MP4 `udta`/`meta` boxes (where phones
    store GPS and device details) are overwritten in place as `free` boxes of the
    same size, so the file still plays and no offsets move.
  - Video length is read from the container (`mvhd` / Matroska `Info/Duration`)
    and enforced on the server (`VIDEO_MAX_SECONDS`, default 30); size by
    `VIDEO_MAX_MB` (default 50). WebM files without a recorded duration are refused.
  - The poster frame is captured **in the browser** (canvas, before upload) and
    validated as an image on the server. There is no transcoding or server-side
    thumbnailing: that would need ffmpeg in the API image (or a queue worker). Until
    then, clips are stored as uploaded, and a video whose codec the viewer's
    browser can't decode shows a clear "can't play here" message.
  - Media is served through an authenticated route that answers Range requests, so
    videos stream and seek. Others only see approved items of active, published,
    unblocked members; owners and staff see everything of their own/reported.
    Videos are click-to-play (`preload="none"`, never autoplay), so feeds load only
    posters.
  - Replacing or deleting an item removes its old files; deleting the main photo
    promotes the next photo.
- **Email** uses a `Mailer` interface. In development it logs links to the server
  console; SMTP is enabled by environment variables.

## 4. Phases

1. Inspection, architecture, design system — tokens, fonts, UI kit.
2. Public pages, auth (register, verify, login, logout, reset), onboarding.
3. Prisma schema, profiles, photos, discovery, likes, matches.
4. Conversations, messages, Socket.IO, block/report/unmatch.
5. Compatibility engine and expert workflow.
6. Curated games (DB-seeded) and Mavelle AI.
7. Admin moderation + audit log, tests, Docker and Lightsail docs.
