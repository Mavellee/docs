# 06 – Frontend guide

The design board is phone-first. We build the same screens as a **responsive website / PWA**:
phones get the board layout; tablets and desktops get the same screens adapted (side navigation,
split panes, wider cards). One codebase, no separate "desktop site".

## Design system

Defined once in `web/src/app/globals.css` (tokens) and `web/src/components/ui` (shadcn components).

| Token | Value | Used for |
|-------|-------|----------|
| `primary` | plum-rose `#8F4A68` | primary buttons, active tab, links |
| `heart` | `#E8527F` | like button, match percentage chips |
| `background` | blush `#FFF7F4` (+ `.bg-blush` gradient) | page background |
| `accent` | `#FBE3EA` | selected chips, icon circles |
| `muted` / `muted-foreground` | soft rose / warm grey | secondary text and surfaces |
| `success` | `#2F9E6F` | verified tick, "approved" |
| Fonts | **Playfair Display** (`font-serif`) for the wordmark and big headings, **Inter** (`font-sans`) for UI | |
| Radius | `--radius: 0.875rem`; buttons are pills (`size="xl"`) | |

Rules: no raw hex in components · no dark mode in the MVP (the board is light) · every interactive element has a visible focus ring and a 44 px minimum touch target · animations respect `prefers-reduced-motion` (already handled globally).

## Route map

Route groups (folders in parentheses) share a layout but do not appear in the URL.

```
src/app/
├── page.tsx                              /                     Splash / landing                 [done]
├── (auth)/
│   ├── sign-up/page.tsx                  /sign-up              phone → OTP, Google              M1
│   ├── sign-in/page.tsx                  /sign-in                                               M1
│   └── verify/page.tsx                   /verify               OTP entry                        M1
├── (onboarding)/                         centered single-column flow with a progress bar
│   ├── onboarding/intent/page.tsx        /onboarding/intent    screen 2                         M2
│   ├── onboarding/profile/page.tsx       /onboarding/profile   screen 3 (+ photos, interests)   M2
│   └── onboarding/astrology/page.tsx     /onboarding/astrology optional birth details + consent M4
├── (app)/                                signed-in shell: bottom tabs (mobile) / sidebar (≥ lg)
│   ├── discover/page.tsx                 /discover             screen 4                         M3
│   ├── matches/page.tsx                  /matches              list with status badges          M3
│   ├── matches/[id]/page.tsx             /matches/:id          screens 5, 6, 7, 11 as tabs/states M3–M4
│   ├── chat/page.tsx                     /chat                 list (desktop: left pane)        M5
│   ├── chat/[conversationId]/page.tsx    /chat/:id             screen 8, 9, 10 (thread pane)    M5–M6
│   ├── events/page.tsx · events/[id]     /events               screen 12                        M7
│   ├── profile/page.tsx                  /profile              my profile preview               M2
│   └── settings/page.tsx                 /settings             screen 13 (tabs: Basic, Photos, Preferences, Privacy) M2
├── (astrologer)/astrologer/…             queue + review workspace (desktop-first)               M4
├── (admin)/admin/…                       moderation, events, astrologers, audit                 M8
└── api/…                                 backend (doc 05)
```

**Guards:** the `(app)` layout redirects: no session → `/sign-in`; session but no `users` row → `/onboarding/intent`; profile incomplete → onboarding step. Role areas check the role on the server (never only in the UI).

## Responsive behaviour

| Breakpoint | Layout |
|-----------|--------|
| `< 640` (phone) | Exactly the board: single column, fixed bottom tab bar (Discover · Matches · Chat · Events · Profile), full-bleed cards. |
| `640–1023` (tablet) | Same as phone, content capped at `max-w-xl` and centred; bottom bar stays. |
| `≥ 1024` (desktop) | Left sidebar (logo, nav, notifications, profile menu). **Discover:** card in the centre, profile details panel on the right instead of a bottom sheet. **Chat:** conversation list left, thread right (split view), Iceberg panel as a collapsible right drawer. **Compatibility:** score ring left, breakdown right. **Astrologer console:** two-column review workspace. |

Build mobile first with Tailwind (`lg:` adds the desktop behaviour). Never hide functionality on small screens.

## Screen-by-screen build spec

Each entry: components → data → states → API.

### 1. Splash / landing  `/`  ✅ built
Hero (logo, wordmark, tagline, two CTAs) + five feature cards that mirror the board's header strip. Fully static (prerendered). CTAs → `/sign-up`, `/sign-in`.

### 2. Intent  `/onboarding/intent`  (M2)
- Three large selectable cards: **Single** (heart), **Situationship** (flame), **Open to Astrology** (moon) – multi-select allowed (`profiles.intent` is an array, min 1). Progress bar (step 1/4). Continue disabled until ≥ 1 selected.
- Data: none to load. Saves via `PATCH /me/profile { intent }`. State: React Hook Form + Zod (`intentSchema`).

### 3. Profile creation  `/onboarding/profile`  (M2)
- Avatar uploader (camera badge) → upload flow below; name; date of birth (age shown, **18+ enforced client and server**); bio with 500-char counter; interest chips from `GET /interests` (max 10); gender; "show me" preference.
- **Photo upload flow:** pick file → client-side resize to max 1600 px + strip EXIF → `POST /me/photos` (signed URL) → `PUT` file to Storage → `POST /me/photos/:id/confirm` → shows a "reviewing" placeholder until `moderation_status = 'approved'`.
- States: saving, field errors from `error.details`, upload failure with retry.

### 4. Discover  `/discover`  (M3)
- Filter chips **All / Compatible / Near you** + filter button (opens sheet: age range, distance, intent).
- **SwipeCard:** large photo, name + age + verified tick, distance ("New Delhi, 3 km away"), match-% chip, sun-sign chip, occupation, interests, photo thumbnails (+N). Drag left = pass, right = like (Framer Motion `drag="x"`), buttons ✕ ★ ♥ do the same (also the keyboard / screen-reader path), undo for the last pass.
- Pre-fetch the next 5 cards and their first photo; optimistic removal of the top card; the like request fires in the background; on `isMatch` show the **Match modal** (screen 7 treatment).
- Empty state: "You've seen everyone nearby" + widen-filters button. Error state: retry.
- Data: `useInfiniteQuery(['discover', filter])` → `GET /discover`. Mutations: `POST /likes`, `POST /passes`.

### 5. Compatibility  `/matches/[id]` tab "Your match"  (M4)
- Score ring (87%) + "High compatibility", four progress rows (Emotional, Communication, Values, Long-term), **Request astrologer review** button (only if both users allowed review), "View detailed report" link → screen 11.
- Tabs **Your Match / Astrologer Review**. Data: `GET /matches/:id`, `GET /matches/:id/compatibility`.

### 6. Under astrologer review  (M4)
- Astrologer card (photo, name, years, rating), a 3-step timeline driven by `match.status` + `astrologer_reviews` (Submitted → Under review "usually within 12–24 h" → Notified). If `awaiting_info`: a form showing the astrologer's question and a reply box → `POST /review-info`. If `declined`: the friendly note + **Appeal** button.
- Auto-refresh: Realtime `notifications` insert for this match invalidates the query.

### 7. Match approved / It's a match  (M3, M4)
- Full-screen overlay: both avatars, heart burst (Framer Motion, ≤ 1.2 s), "Approved by our astrologers" (only when a review happened), note from astrologer, **Start chatting** → `/chat/:id`, "View compatibility report". Confetti is disabled under reduced-motion.

### 8. Chat  `/chat/[conversationId]`  (M5)
- Header (avatar, name, online dot from presence, call icons hidden until a later phase), message list (virtualised once > 200 messages), grouped by day, bubble tails, delivery/read ticks, composer with emoji/attach/voice, typing indicator (broadcast).
- Data flow: `useInfiniteQuery` for history + Realtime subscription that **patches** the same cache; send = optimistic bubble with client UUID, replaced by the server row; failed sends show "Retry".
- Locked states: match not yet approved → "Waiting for astrologer approval" screen instead of the composer; conversation `closed_at` → read-only banner.

### 9. Iceberg AI  (M6)
- Inside the chat: Iceberg posts a **PromptCard** message ("Here's a question for you both"). Tapping opens the answer panel: question, text box, **Share answer** / **Skip** (no reason needed), and "Next question in 02:15:33" countdown from `next_prompt_at`. Answers stay hidden until both have answered (or skipped).
- Per-conversation controls: pause 24 h, turn off, max depth (light → deeper). The user's global ceiling (`iceberg_max_level`) always wins.

### 10. Games  (M6)
- Games sheet with the five cards from the board and **All games / Your progress** tabs. Each game is a self-contained component implementing one interface: `render(session, myId)` + `onMove(move)`. State is the server's `game_sessions.state`; Realtime updates both players. Start with **This or That** and **Would You Rather** (simplest: both pick, then reveal), then Two Truths & a Lie, Guess Each Other, Emoji Story.

### 11. Detailed compatibility report  (M4)
- Tabs **Astrology / Personality / Interests**: ring + the four rows, then a short plain-language explanation per row. Wording rule: insights, never guarantees.

### 12. Campus events  `/events`  (M7)
- Event cards (cover, title, date/venue, "registered" / "waitlisted" / Register). Detail page adds description, capacity, "post-event matching" opt-in. After registering: ticket with **QR code** (`qrcode`), add-to-calendar. Staff check-in screen with camera scan (`html5-qrcode`).

### 13. Edit profile and settings  `/settings`  (M2)
- Tabs **Basic info / Photos / Preferences**: "Show me people who are" (Single / Situationship / Both), age range slider, preferred interests chips, toggles **Show my astrology details** and **Allow astrologers to review my matches** (consent rows are written when toggled), Iceberg level, notification toggles, block list, delete account (30-day grace, typed confirmation).

## State and data rules

| Kind of state | Tool | Example |
|---------------|------|---------|
| Data from the server | **TanStack Query** | feed, matches, messages, settings |
| Form state | **React Hook Form** + Zod resolver | onboarding, settings |
| Ephemeral UI state | `useState` first; **Zustand** only when shared across distant components | filter sheet open, swipe-deck undo stack |
| URL state | search params | `?filter=near`, selected tab |
| Never | copying server data into Zustand or `useEffect` fetching | |

### Query key convention
`[resource, scope, params]` → `['discover', 'feed', {filter}]`, `['matches', 'detail', id]`, `['messages', conversationId]`. After a mutation, **invalidate by prefix** (`['matches']`) or patch the cache (`setQueryData`) for instant UI.

### Feature folders
```
src/features/<feature>/
├── api.ts          typed functions using apiFetch + the shared Zod types
├── hooks.ts        useDiscoverFeed(), useLike() ... (TanStack Query)
├── components/     UI used only by this feature
└── index.ts        the public surface other code may import
```
Shared, feature-agnostic building blocks go in `src/components` (`ui/` = shadcn, `brand/`, `layout/`).

## Next.js 16 specifics to remember (this project has *Cache Components* enabled)

- Reading the session (`cookies()`/`headers()`) happens at **request time**, so any component that does it must sit under a `<Suspense>` boundary, or the build tells you to add one. Pattern: the page/layout is a fast static shell; the personalised part is a child inside `<Suspense fallback={<Skeleton/>}>`.
- Public pages (landing, event detail for logged-out users) stay fully static → fast and cheap.
- Interactive screens (swipe deck, chat) are **client components** that fetch through TanStack Query; the server component above them only passes ids.
- `middleware` is called **`proxy.ts`** in this Next version; it refreshes the Supabase session cookie and does coarse redirects (M1).
- Read the bundled docs in `web/node_modules/next/dist/docs/` when unsure – `web/AGENTS.md` explains why; this version differs from older tutorials.

## Accessibility and quality bar

- Semantic HTML first (`button`, `nav`, `main`, headings in order); shadcn/Base UI components give correct ARIA for dialogs, tabs, sheets.
- Every image has `alt` (profile photos: "Photo of {name}"); icons that carry meaning have labels.
- Colour contrast ≥ 4.5:1 for text (check `primary` on `background` when changing tokens).
- Swipe is never the only way: buttons and keyboard (←/→/↑) work.
- Lighthouse targets on `/`: Performance ≥ 90, Accessibility ≥ 95; app screens: LCP < 2.5 s on a mid-range phone over 4G.
- Images via `next/image`, sized, `priority` only for the first card. Photos come from signed URLs → configure `images.remotePatterns` for the Supabase host in M2.
