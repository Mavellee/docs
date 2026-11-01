# 01 – Product and scope

## The product in one paragraph

Iceberg helps people (initially college students and young professionals in India) find a *meaningful*
connection rather than endless swiping. Users declare an **intent** (Single / Situationship / Open to
Astrology). When two people like each other they get a **match**. If *both* opted in, the match is
first reviewed by a human **astrologer** before chat unlocks. Inside the chat an AI companion called
**Iceberg** drops conversation prompts and mini-games so that people actually talk. Campus **events**
connect the online and offline worlds.

## Personas

| Persona | Role in DB (`user_role`) | What they do |
|---------|--------------------------|--------------|
| Member | `user` | Builds a profile, discovers, likes, chats, plays games, registers for events |
| Astrologer | `astrologer` | Claims matches from a queue and approves / asks for more info / declines |
| Moderator | `moderator` | Works the reports queue, reviews flagged photos and messages |
| Admin | `admin` | Vets astrologers, manages events and colleges, sees audit log |

## Screen → feature map (from the design board)

The design board has 13 phone screens. For the website we keep the same screens and flows, but
adapt the layout for desktop (see doc 06).

| # | Screen | Feature | DB tables / functions it touches | Milestone |
|---|--------|---------|----------------------------------|-----------|
| 1 | Splash | Marketing landing + sign-in / sign-up entry | – | M1 |
| 2 | Onboarding – intent | Choose Single / Situationship / Open to Astrology | `profiles.intent` | M2 |
| 3 | Profile creation | Photo, name, age, bio, interests | `users`, `profiles`, `profile_photos`, `profile_interests` | M2 |
| 4 | Discover | Card stack, filters (All / Compatible / Near You), ✕ ★ ♥ | `discover_profiles()`, `like_user()`, `pass_user()`, `saved_profiles` | M3 |
| 5 | Astrology compatibility | Score ring, 4-part breakdown, "Request astrologer review" | `compatibility_scores`, `matches.score_breakdown`, `astrology_details` | M4 |
| 6 | Under astrologer review | Status timeline (submitted → under review → notified) | `astrologer_reviews`, `notifications` | M4 |
| 7 | Match approved | "It's a Match!" celebration → Start chatting | `matches`, `create_conversation_for_match()` | M3/M4 |
| 8 | Chat | Real-time 1-to-1 chat, typing, read receipts | `conversations`, `messages`, `conversation_participants` | M5 |
| 9 | Iceberg AI | Prompt card, share answer / skip, next-question countdown | `prompts`, `conversation_prompts`, `prompt_responses` | M6 |
| 10 | In-chat games | This or That, Guess Each Other, Would You Rather, Two Truths & a Lie, Emoji Story | `game_sessions`, `game_moves` | M6 |
| 11 | Compatibility insights | Detailed report: Astrology / Personality / Interests tabs | `compatibility_scores`, `matches` | M4 |
| 12 | Campus events | Event cards, register, QR check-in | `events`, `event_registrations` | M7 |
| 13 | Preferences and settings | Edit profile, show-me, age range, privacy toggles | `profiles`, `user_settings` | M2 |

## MVP scope

**In the MVP**

- Phone-OTP and Google sign-in, 18+ age gate, consent capture
- Profile + photos (private storage, moderated), interests, preferences
- Discover feed with distance / intent / mutual-preference filtering
- Like / pass / save / block / report / unmatch
- Mutual-like → match; optional astrologer review; astrologer console
- Real-time chat (text first; voice notes and images after)
- Iceberg prompts (levels 1–2 from the curated library, levels 3–4 via LLM) and 5 games
- Campus events with registration, waitlist and QR check-in
- Notifications (in-app first, then web push and email)
- Account deletion with 30-day purge, data export

**Explicitly not in the MVP** (write them down so we don't drift)

- Native iOS/Android apps (the PWA covers this; the API is app-ready if we add them later)
- Payments / subscriptions
- Video calls
- Real ephemeris-based chart calculation (MVP: astrologer judges from birth details; score is a placeholder algorithm, see doc 05)
- Multi-language UI (English only; strings go through one file so we can add Hindi later)

## Success metrics (what PostHog will track)

| Metric | Event(s) |
|--------|----------|
| Profile completion rate | `profile_completed` / `signup_completed` |
| Like → match rate | `like_sent`, `match_created` |
| Match → first message (chat conversion) | `match_created`, `first_message_sent` |
| Iceberg prompt engagement | `prompt_shown`, `prompt_answered`, `prompt_skipped` |
| Astrologer SLA | `review_submitted`, `review_completed` (time delta; target < 24 h) |
| Event registrations | `event_registered`, `event_checked_in` |

## Open product questions (decide before the milestone that needs them)

| Question | Needed by |
|----------|-----------|
| Is astrologer review *free*, *capped per user*, or paid later? (schema has `review_requirement`) | M4 |
| What exactly does the compatibility score use for the MVP? (Doc 05 proposes a simple, explainable formula) | M4 |
| Do we verify a college email at sign-up or only for event access? | M2 |
| LLM vendor for Iceberg levels 3–4 (Claude vs OpenAI) and monthly cost cap | M6 |
