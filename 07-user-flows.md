# 07 – User flows

Sequence diagrams for every flow that crosses more than one component. They render on GitHub and in
VS Code (Markdown Preview Mermaid Support extension). When code and diagram disagree, fix whichever is wrong in the same PR.

## 1. Sign-up and first-time setup (M1–M2)

```mermaid
sequenceDiagram
  actor U as User
  participant W as Web app
  participant A as Supabase Auth
  participant API as /api/v1
  participant DB as Postgres

  U->>W: enter phone number
  W->>API: (rate-limit check: 3 OTPs / 10 min / phone, 10 / h / IP)
  W->>A: signInWithOtp({phone})
  A-->>U: SMS code (MSG91)
  U->>W: enter code
  W->>A: verifyOtp
  A-->>W: session (httpOnly cookie)
  W->>API: GET /me
  API-->>W: 404 "no profile yet"
  W->>U: onboarding: intent → profile
  U->>W: DOB + terms + privacy consent
  W->>API: POST /me/bootstrap
  API->>DB: insert users(id = auth uid, dob) → trigger enforce_min_age
  API->>DB: insert consents (terms, privacy)
  DB-->>API: ok  (trigger created user_settings row)
  API-->>W: 201
  U->>W: profile fields, interests, photos
  W->>API: PATCH /me/profile, PUT /me/interests, photo upload flow
  API->>DB: update profiles, recompute completeness
  W->>U: → /discover
```

Failure cases to handle: wrong/expired OTP (attempt limit), under-18 DOB (400 `validation_failed`, friendly message, nothing stored), duplicate phone (merge into the existing account), SMS provider outage (fallback to email magic link).

## 2. Photo upload with moderation (M2)

```mermaid
sequenceDiagram
  actor U as User
  participant W as Web app
  participant API as /api/v1
  participant S as Supabase Storage
  participant M as Image moderation
  participant DB as Postgres

  U->>W: choose photo
  W->>W: resize ≤1600px, strip EXIF, JPEG/WebP
  W->>API: POST /me/photos
  API->>DB: insert profile_photos(status = pending, position)
  API->>S: createSignedUploadUrl(photos/{userId}/{photoId})
  API-->>W: {photoId, uploadUrl}
  W->>S: PUT file
  W->>API: POST /me/photos/{id}/confirm
  API->>M: analyse (nudity, violence, minors)
  alt clean
    API->>DB: moderation_status = approved
  else flagged
    API->>DB: moderation_status = rejected (+ queue for human if borderline)
  end
  API-->>W: status
  Note over W: Others only ever see signed URLs of approved photos
```

## 3. Discover → like → match (M3)

```mermaid
sequenceDiagram
  actor A as Aanya
  participant W as Web app
  participant API as POST /likes
  participant DB as like_user()
  actor B as Arjun

  A->>W: swipes right on Arjun
  W->>W: optimistic: remove card, show next
  W->>API: {toUserId: Arjun}
  API->>DB: select * from like_user(Aanya, Arjun)
  Note over DB: advisory lock on the pair, block check, active check,<br/>insert like, delete pass, look for reverse like
  alt Arjun has NOT liked Aanya
    DB-->>API: is_match = false
    API-->>W: {isMatch:false}
  else Arjun already liked Aanya
    DB->>DB: insert matches (user_a < user_b)
    alt both allowed astrologer review
      DB->>DB: status = pending_review, insert astrologer_reviews (due +24h)
      DB->>DB: notify both: "An astrologer is reviewing your compatibility"
    else
      DB->>DB: status = approved, create conversation + participants
      DB->>DB: notify both: "It's a match!"
    end
    DB-->>API: is_match = true, match_id
    API-->>W: {isMatch:true, matchId}
    W->>A: It's a Match! overlay
  end
  Note over B: Arjun gets a notification (in-app now, push in M8)
```

Why the lock matters: if Aanya and Arjun like each other in the same millisecond, without the per-pair advisory lock *both* transactions could see "no reverse like yet" and the match would never be created.

## 4. Astrologer review lifecycle (M4)

```mermaid
stateDiagram-v2
  [*] --> pending_review: mutual like + both opted in
  [*] --> approved: mutual like, no review needed
  pending_review --> approved: astrologer approves
  pending_review --> awaiting_info: astrologer needs more info
  pending_review --> declined: astrologer declines
  awaiting_info --> pending_review: users reply (new attempt, new 24h clock)
  declined --> pending_review: appeal overturned
  approved --> unmatched: either user unmatches / blocks
  pending_review --> unmatched: either user unmatches / blocks
  awaiting_info --> expired: no reply in 7 days (job)
  approved --> [*]
  unmatched --> [*]
  declined --> [*]
  expired --> [*]
```

```mermaid
sequenceDiagram
  actor Ast as Astrologer
  participant W as Console
  participant API as /api/v1/astrologer
  participant DB as Postgres

  Ast->>W: "Take next review"
  W->>API: POST /reviews/claim
  API->>DB: claim_next_review(astrologer)
  Note over DB: oldest unassigned, FOR UPDATE SKIP LOCKED,<br/>respects max_open_reviews
  DB-->>API: review_id | null
  W->>API: GET /reviews/{id}
  API->>DB: select from astrologer_review_view
  Note over API,DB: only names, ages, birth details, score. No photos, phone, email, chat.
  Ast->>W: checklist + decision + note to users
  W->>API: POST /reviews/{id}/outcome
  API->>DB: submit_review_outcome(...)
  DB->>DB: complete review, update match, create conversation if approved,<br/>notify both users, write audit_log
```

## 5. Chat with live updates (M5)

```mermaid
sequenceDiagram
  actor A as Aanya
  participant WA as Aanya's browser
  participant API as POST /messages
  participant DB as Postgres
  participant RT as Supabase Realtime
  participant WB as Arjun's browser
  actor B as Arjun

  A->>WA: type + send
  WA->>WA: optimistic bubble (client UUID, "sending")
  WA->>API: {id: uuid, body}
  API->>API: participant? conversation open? blocked? text moderation, rate limit
  API->>DB: insert messages (on conflict id do nothing)
  DB->>DB: trigger bump_conversation_on_message
  API-->>WA: saved row → bubble becomes "sent"
  DB-->>RT: INSERT event (RLS: only participants receive it)
  RT-->>WB: new message
  WB->>WB: patch cache, mark delivered
  B->>WB: opens thread
  WB->>API: POST /read
  API->>DB: last_read_message_id
  RT-->>WA: ticks turn blue (if read receipts on for both)
  Note over WA,WB: typing + online = broadcast / presence only, never stored
```

## 6. Iceberg prompt cycle (M6)

```mermaid
sequenceDiagram
  participant Cron as pg_cron (every 5 min)
  participant EF as Edge Function
  participant DB as Postgres
  participant LLM as LLM API
  actor U as Both users

  Cron->>EF: tick
  EF->>DB: conversations where iceberg on, not paused,<br/>next_prompt_at <= now(), both active recently
  loop each due conversation
    EF->>EF: pick level = min(conversation max, both users' max, progression by days/messages)
    alt level light or playful
      EF->>DB: pick unused approved prompt from library (no AI cost)
    else level values or deeper
      EF->>LLM: generate from tags + shared interests (system prompt forbids sexual/medical/legal/self-harm)
      EF->>EF: moderation check on output → fall back to library if flagged
    end
    EF->>DB: insert message (type ai_prompt) + conversation_prompts, set next_prompt_at (+ jitter)
  end
  DB-->>U: realtime: prompt card appears
  U->>DB: respond / skip → prompt_responses
  Note over U: Answers revealed only once both answered or skipped
```

Cost guards: daily budget per environment, max 1 LLM call per conversation per day, circuit breaker → library fallback.

## 7. Event registration, waitlist and QR check-in (M7)

```mermaid
sequenceDiagram
  actor U as User
  participant API as /events
  participant DB as Postgres
  actor S as Staff
  participant Scan as Check-in screen

  U->>API: POST /events/{id}/register
  API->>DB: lock event row; count registered
  alt capacity free
    DB-->>API: status = registered, qr_token
  else full
    DB-->>API: status = waitlisted, waitlist_position
  end
  API-->>U: ticket (QR of signed token)
  Note over U: cancel → job promotes waitlist #1 and notifies
  S->>Scan: scan QR at the door
  Scan->>API: POST /events/check-in {token}
  API->>DB: update ... set status = checked_in where qr_token = $1 and status = registered (single use)
  API-->>Scan: ✅ name / ❌ already used / ❌ wrong event
```

## 8. Safety: block, report, unmatch (M8)

```mermaid
flowchart TD
  A[User taps ⋯ on profile or message] --> B{Choice}
  B -->|Unmatch| C["unmatch(): match = unmatched, chat closed, open review closed"]
  B -->|Block| D["block_user(): insert block + unmatch if matched"]
  B -->|Report| E["insert reports row (+ message snapshot reference)"]
  D --> F[Blocked person disappears from Discover both ways, cannot like/message]
  E --> G[Moderator queue, sorted by reason severity]
  G --> H{Decision}
  H -->|warn / mute| I[moderation_actions row + notification]
  H -->|suspend / ban| J["users.status changes; sessions revoked; audit_log"]
  H -->|dismiss| K[report closed with note]
```

Hard rules: reports about `underage` or `threat` are escalated immediately and the reported account is auto-suspended pending review.

## 9. Account deletion and data export (M8)

```mermaid
sequenceDiagram
  actor U as User
  participant API as /me
  participant DB as Postgres
  participant Cron as daily job

  U->>API: DELETE /me (typed confirmation)
  API->>DB: request_account_deletion(user)
  Note over DB: deletion_requests row (purge_after = +30 days),<br/>users.status = deactivated, deleted_at = now()
  API->>API: revoke sessions, remove from Discover immediately
  API-->>U: "Your account is scheduled for deletion on {date}. Log in before then to cancel."
  Cron->>DB: purge_deleted_accounts()
  Note over DB: hard delete users → cascades to profiles, photos rows, likes, messages, ...
  Cron->>Cron: delete storage objects for those users (photos, voice)
```

Open item for M8: storage objects are not removed by the SQL cascade, so the purge job must also delete `photos/{userId}/*`, `voice-intros/{userId}/*`, `chat-media/*` for the purged ids.
