# Knot API

Base path `/api`. JSON in and out.

- **Success:** `{ "data": ... }` with 200/201.
- **Error:** `{ "error": { "code": "VALIDATION_FAILED", "message": "…", "details": { "field": ["…"] } } }`
  with 400 (bad request), 401 (not signed in), 403 (forbidden / bad origin), 404 (not found or
  not yours — ids are never confirmed to non-members), 409 (conflict), 422 (validation), 429
  (rate limited), 500.
- **Auth:** `knot_session` HTTP-only cookie set by register/login.
- **CSRF:** non-GET requests must send an `Origin` (or `Referer`) equal to `APP_ORIGIN`.
- **Access levels:** _public_, _auth_ (signed in, any status), _active_ (verified & not restricted),
  _expert_ (approved expert), _staff_ (moderator/admin), _admin_.

## Auth `/auth`
| Method | Path | Access | Notes |
|---|---|---|---|
| POST | `/register` | public | `{email,password,dateOfBirth:"YYYY-MM-DD",ageConfirmed:true,acceptTerms:true}`; 422 `UNDERAGE` if under 18 |
| POST | `/login` | public | Same error for unknown email and wrong password |
| POST | `/logout` | auth | Revokes the session server-side |
| GET | `/me` | auth | Own account view |
| POST | `/verify-email` | public | `{token}`; single use, 24 h |
| POST | `/resend-verification` | auth | |
| POST | `/forgot-password` | public | Always 200 |
| POST | `/reset-password` | public | `{token,password}`; revokes all sessions |
| POST | `/change-password` | auth | `{currentPassword,newPassword}`; revokes other sessions |

## Profiles & reference data
| Method | Path | Access | Notes |
|---|---|---|---|
| GET/PATCH | `/profiles/me` | auth | Partial update: basics, `approxLocation` (rounded), languages, `interests` (slugs), `intentions`, `soughtIntentions`, age range, distance |
| POST | `/profiles/me/publish` | auth | Requires name, gender, interested-in, intention, ≥1 photo |
| POST | `/profiles/me/photos` | auth | multipart `photo`; JPEG/PNG/WebP by magic bytes, ≤5 MB, ≤6 photos, metadata stripped |
| DELETE | `/profiles/me/photos/:id` | auth | |
| PUT | `/profiles/me/photos/order` | auth | `{ids:[...]}` |
| GET | `/profiles/:userId` | active | Public view + compatibility explanation; 404 if blocked/unpublished |
| GET | `/photos/:id` | auth | Streams a photo; 404 between blocked users |
| GET | `/meta/interests`, `/meta/cities` | public | |

## Discovery, likes, matches
| Method | Path | Access | Notes |
|---|---|---|---|
| GET | `/discovery?cursor&limit` | active | `{items,nextCursor}`; two-way preferences, excludes self/blocked/acted-on |
| POST | `/likes` | active | `{userId}` → `{matched,matchId,alreadyLiked}` (201 new, 200 repeat) |
| POST | `/likes/pass` | active | `{userId}` |
| GET | `/likes/received`, `/likes/sent` | active | |
| GET | `/matches` | active | Active matches |
| DELETE | `/matches/:id` | active | Unmatch (closes chat and shared games) |

## Conversations
| Method | Path | Access | Notes |
|---|---|---|---|
| GET | `/conversations` | active | With unread counts |
| GET | `/conversations/:id` | active | Member + active match + not blocked |
| GET | `/conversations/:id/messages?before&limit` | active | Newest page first, `before` = message id |
| POST | `/conversations/:id/messages` | active | `{body,clientId?}`; ≤2000 chars; 30/min |
| POST | `/conversations/:id/attachments` | active | multipart `image` |
| GET | `/conversations/:id/attachments/:messageId` | active | |
| POST | `/conversations/:id/read` | active | Emits read receipt if enabled |

**Socket.IO** (`/socket.io`, same cookie). Client → server (with ack `{ok,data|error}`):
`conversation:join`, `conversation:leave`, `message:send {conversationId,body,clientId}`,
`typing {conversationId,isTyping}`, `conversation:read`. Server → client: `message:new`,
`typing`, `conversation:read`, `conversation:updated`, `match:new`, `match:ended`,
`notification`, `game:update`. Every handler re-validates the session and membership.

## Compatibility `/compatibility`
`GET /questions` · `PUT /answers/:questionId {answer,acceptable[],importance 0-3,isDealbreaker}` ·
`DELETE /answers/:questionId` · `GET /with/:userId` (active) · `GET /method`.

## Experts `/experts` (active)
Members: `POST /apply`, `GET /application`, `GET /` (approved experts), `POST /requests
{topic,question,share:{profile,intentions,compatibilityAnswers,matchId?},consent:true}`,
`GET /requests`, `POST /requests/:id/respond`, `POST /requests/:id/cancel`.
Experts: `GET /workspace/queue`, `GET /workspace/history`, `GET /workspace/queue/:id`,
`POST /workspace/queue/:id/claim|request-info|complete|decline {message}`.

## Games `/games` (active)
`GET /` · `POST /sessions {gameSlug,matchId?,questionCount}` · `GET /sessions` ·
`GET /sessions/:id` · `POST /sessions/:id/answer {answer}` · `/skip` · `/next` (shared) · `/end`.

## Knot AI `/ai` (active)
`POST /suggest` with `feature` one of `conversation_starter {matchId}`, `icebreaker {userId}`,
`bio_help {draft}`, `reflection`, `first_date {matchId}`, `explain_compatibility {userId}` →
`{source:"ai"|"curated",label,suggestions[],note?}`. `GET /status`.

## Safety, notifications, settings
`/safety`: `GET/POST /blocks`, `DELETE /blocks/:userId`, `POST /reports
{userId,target:USER|PROFILE|MESSAGE,messageId?,reason,details?,alsoBlock?}`, `GET /reports`.
`/notifications`: `GET /`, `POST /:id/read`, `POST /read-all`.
`/settings`: `GET/PATCH /`, `POST /discovery {paused?,isDiscoverable?}`, `GET /sessions`,
`DELETE /sessions/:id`, `GET /export`, `POST /delete-account {password}`.

## Admin `/admin` (staff; audit log admin-only)
`GET /overview`, `GET /reports?status`, `POST /reports/:id {status,resolution?}`,
`GET /users?q`, `POST /users/:id/status {status,reason}`, `POST /photos/:id/reject {reason}`,
`GET /experts?status`, `POST /experts/:id/decision {decision,note,credentialsVerified}`,
`GET /audit-log`. Every mutating admin action writes to the append-only `AdminAuditLog`.
