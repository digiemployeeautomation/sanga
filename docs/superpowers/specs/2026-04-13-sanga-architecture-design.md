# Sanga -- Architecture Design Spec

**Version:** 1.0
**Date:** 2026-04-13
**Status:** Approved

A location-based dating app built Supabase-native with Expo (React Native) for mobile and Next.js for a landing page. Mobile-first MVP.

---

## 1. Architecture Overview

Supabase-native, mobile-first. No custom backend server. Supabase client SDK talks directly from the Expo app, secured by Row Level Security (RLS). Edge Functions handle server-side logic only where client-side is insufficient.

```
┌─────────────┐     ┌──────────────────────────────────────────┐
│  Expo App   │────▶│              Supabase                     │
│  (Mobile)   │     │  ┌──────────┐  ┌───────────┐  ┌────────┐│
└─────────────┘     │  │   Auth   │  │  Database  │  │Storage ││
                    │  │(Phone,   │  │(Postgres + │  │(Photos)││
┌─────────────┐     │  │ Google,  │  │  PostGIS)  │  │        ││
│  Next.js    │     │  │ Apple)   │  │            │  │        ││
│  (Landing)  │     │  └──────────┘  └───────────┘  └────────┘│
└─────────────┘     │  ┌──────────┐  ┌───────────┐            │
       │            │  │ Realtime │  │   Edge     │            │
       ▼            │  │(Chat,    │  │ Functions  │            │
    Vercel          │  │ Presence)│  │(Matching,  │            │
                    │  │          │  │ Moderation)│            │
                    │  └──────────┘  └───────────┘            │
                    └──────────────────────────────────────────┘
                                       │
                                       ▼
                              ┌──────────────────┐
                              │ Expo Notifications│
                              │   (FCM + APNs)    │
                              └──────────────────┘
```

### Tech Stack

| Layer | Technology | Replaces (from constitution) |
|---|---|---|
| Mobile App | Expo (React Native) + TypeScript | React Native CLI |
| Web (Landing) | Next.js + TypeScript | Same |
| Database | Supabase Database (Postgres + PostGIS) | PostgreSQL + PostGIS |
| Auth | Supabase Auth (Phone OTP, Google, Apple) | Firebase Auth |
| Storage | Supabase Storage (photos, media) | AWS S3 |
| Realtime Chat | Supabase Realtime (Broadcast + Presence) | Socket.io |
| Backend Logic | Supabase Edge Functions (Deno/TS) | Node.js + Express |
| Sessions | Supabase Auth (JWT, auto-refresh) | Redis |
| Push Notifications | Expo Notifications (FCM + APNs) | Firebase Cloud Messaging |
| Web Hosting | Vercel | AWS |
| CI/CD | GitHub Actions | Same |
| Monitoring | Sentry | Same |
| CDN | Supabase Storage CDN + Vercel Edge | Cloudflare |

---

## 2. Data Model & Row Level Security

All tables in Supabase Postgres with PostGIS enabled. Supabase Auth manages `auth.users` -- our `profiles` table references it via `id = auth.uid()`.

### Core Tables

| Table | Key Fields | RLS Policy |
|---|---|---|
| `profiles` | id (= auth.uid()), name, bio, dob, gender, location (geometry), avatar_url, is_verified, created_at | Users can read others' profiles (filtered by discovery); can only update own |
| `photos` | id, user_id, url, position, is_verified, created_at | Owner can CRUD own (max 6 enforced by Edge Function); others can read |
| `preferences` | user_id (PK), min_age, max_age, gender_pref, max_distance_km | Owner only -- read/write own row |
| `swipes` | id, swiper_id, swipee_id, action (like/pass/super), created_at | Insert own swipes only; no read access (matching handled by Edge Function) |
| `matches` | id, user1_id, user2_id, matched_at, unmatched_at | Read only if you're user1 or user2; Edge Function creates matches |
| `messages` | id, match_id, sender_id, body, sent_at, read_at | Read/write only if you're part of the match |
| `reports` | id, reporter_id, reported_id, reason, details, status, created_at | Insert own reports; no read access (admin only) |
| `subscriptions` | id, user_id, plan, started_at, expires_at, is_active | Read own; write via webhook Edge Function only |

### Client-side via RLS (no backend needed)
- Profile reads/updates
- Photo management (CRUD with position ordering)
- Preference updates
- Message sending/reading within matches
- Report submission

### Server-side via Edge Functions (trust required)
- `swipe` -- records swipe, checks mutual like, creates match, triggers push
- `discover` -- PostGIS distance queries, preference filtering, ranking
- `moderate-photo` -- AI moderation on Storage upload
- `push-notify` -- Expo Push API dispatch

---

## 3. Realtime Chat & Presence

Chat uses Supabase Realtime Broadcast channels, one per match.

- **Channel naming**: `match:{match_id}`
- **Messages**: Written to `messages` table via client SDK (RLS allows if you're in the match). A Postgres trigger broadcasts the new message to the Realtime channel so the other user gets it instantly.
- **Typing indicators**: Realtime Presence on the match channel -- each user tracks `{typing: true/false}`. No database writes needed.
- **Read receipts**: Client updates `messages.read_at` via SDK when messages are viewed. Broadcast notifies sender.
- **Online status**: Realtime Presence on a global `online` channel (Phase 2, optional).

No Socket.io needed -- Supabase Realtime runs over WebSockets with automatic reconnection, auth token refresh, and channel subscriptions.

---

## 4. Storage & Photo Management

Supabase Storage with one bucket, RLS-secured.

### Bucket structure
```
photos/
  {user_id}/
    profile/
      0.jpg    # position 0 (primary photo)
      1.jpg    # position 1
      ...      # max 6
    verification/
      selfie.jpg   # Phase 2: verification selfie
```

### Upload flow
1. User picks photo in Expo (via `expo-image-picker`)
2. Image compressed client-side (max 2MB, JPEG/WebP)
3. Uploaded to Supabase Storage via client SDK -- RLS ensures users can only write to their own `{user_id}/` path
4. Storage `on_insert` webhook triggers `moderate-photo` Edge Function
5. Edge Function calls AI moderation API (e.g., Sightengine or Google Vision)
6. If approved, inserts row into `photos` table with public URL. If rejected, deletes from storage and notifies user.

### Storage RLS
- `INSERT`: `auth.uid() = path_tokens[1]` (own folder only)
- `SELECT`: public read for profile photos
- `DELETE`: `auth.uid() = path_tokens[1]` (own folder only)

### Constraints
- Max 6 profile photos (Edge Function checks count)
- Max 10MB per image (Storage bucket config)
- JPEG, PNG, WebP only (Storage bucket config)
- CDN: Supabase Storage built-in CDN with image transformations for thumbnails

---

## 5. Authentication Flow

Supabase Auth replaces Firebase Auth entirely.

### Sign-up flow
1. User opens app, chooses Phone, Google, or Apple sign-in
2. **Phone (primary)**: Supabase Auth sends SMS OTP, user enters code, session created
3. **Google/Apple**: OAuth redirect via Expo AuthSession, Supabase exchanges token, session created
4. On first login, a database trigger (`on_auth.user_created`) inserts a row into `profiles` with defaults
5. User lands on profile creation screen (name, photos, bio, dob, gender, preferences)

### Session management
- JWT access tokens (1 hour expiry) + refresh tokens (auto-rotated)
- `supabase-js` SDK handles token refresh automatically
- Session persisted via `expo-secure-store` (encrypted on-device)
- No Redis needed

### Security
- Phone verification required for all accounts
- Rate limiting: Supabase Auth built-in rate limits on OTP
- API rate limiting: Edge Functions enforce per-endpoint limits (swipes: 100/hr, messages: 200/hr)
- All traffic over HTTPS
- Age >= 18 enforced by Edge Function at profile completion
- RLS policy prevents setting `dob` to a date making user < 18

---

## 6. Edge Functions

Six Edge Functions in TypeScript (Deno runtime).

| Function | Trigger | What it does |
|---|---|---|
| `discover` | HTTP (app) | PostGIS distance query, preference filters, excludes swiped/blocked, ranked feed. Phase 1: recency + random. Phase 2: ELO-like scoring. |
| `swipe` | HTTP (app) | Records swipe, mutual like check, creates match, triggers push. Rate limited 100/hr. |
| `moderate-photo` | Storage webhook | AI moderation API call, approve/reject, update `photos` table or delete. |
| `push-notify` | Internal (other functions) | Expo Push API dispatch. Handles match, message, super like. Respects preferences and 3/day cap. |
| `complete-profile` | HTTP (app) | Validates age >= 18, required fields, marks profile active. Runs once after signup. |
| `report` | HTTP (app) | Creates report, auto-triage. 3+ confirmed reports = auto-suspend. Urgent keywords flagged for escalation. |

### Client-side (no Edge Function)
- Profile reads/updates, messages, photo uploads, preference updates, blocking/unmatching

---

## 7. Push Notifications & Engagement

Expo Notifications bridges the FCM/APNs gap.

### Setup
- `expo-notifications` in the Expo app
- On login, register push token and store in `push_tokens` table (user_id, token, platform)
- Edge Functions send via Expo Push API (`https://exp.host/--/api/v2/push/send`)

### Triggers

| Event | Method | Timing |
|---|---|---|
| New match | Push | Immediate |
| New message | Push | Batched if multiple in 60s |
| Super Like received | Push | Immediate |
| Profile boost active | In-app only | On app open |
| Inactive 3 days | Push | "People are waiting to meet you" |
| Inactive 7 days | Email (Supabase Auth) | Re-engagement with match suggestions |

### Implementation
- Real-time: `push-notify` Edge Function called by `swipe` or Postgres trigger on `messages` insert
- Re-engagement: CRON Edge Function runs daily, queries `last_active_at` thresholds
- Preferences: `notification_preferences` table checked before sending
- Cap: max 3 push per day per user (tracked in `notification_log` table)

---

## 8. Monetization & Subscriptions

Stripe for web, RevenueCat for in-app purchases (App Store/Play Store).

### Tiers

| Tier | Price | Enforced by |
|---|---|---|
| Free | $0 | RLS + Edge Functions (50 daily swipes, basic filters) |
| Plus | $9.99/mo | `subscriptions` table checked by RLS/Edge Functions |
| Gold | $19.99/mo | Same, with additional feature flags |

### Flow
- `subscriptions` table: user_id, plan, started_at, expires_at, is_active
- Mobile: RevenueCat SDK handles App Store/Play Store billing, webhook calls `subscription-webhook` Edge Function, updates `subscriptions` table
- Feature gating: `useSubscription()` hook reads plan, controls UI

### Premium features

| Feature | Gating mechanism |
|---|---|
| Unlimited swipes | `swipe` function checks plan before 50/day limit |
| 5 Super Likes/day | Counter in `daily_usage` table, CRON reset |
| 1 Boost/month | `boosts` table, `discover` function checks |
| Passport | `discover` uses custom location if Plus+ |
| See Who Liked You | RLS on `swipes` allows read if Gold |
| Undo last swipe | `undo-swipe` Edge Function, Plus+ only |

---

## 9. Monorepo Structure

Turborepo with npm workspaces.

```
sanga/
├── apps/
│   ├── mobile/                # Expo app (React Native)
│   │   ├── app/               # Expo Router file-based routing
│   │   ├── components/        # Mobile UI components
│   │   ├── hooks/             # useAuth, useSubscription, useChat, etc.
│   │   ├── lib/               # Supabase client init, helpers
│   │   ├── assets/            # Icons, images, fonts
│   │   ├── app.json           # Expo config
│   │   └── package.json
│   └── web/                   # Next.js landing page
│       ├── app/               # App Router
│       ├── components/        # Landing page components (shadcn/ui)
│       └── package.json
├── packages/
│   ├── shared/                # Cross-platform code
│   │   ├── types/             # Database types (generated by Supabase CLI)
│   │   ├── validation/        # Zod schemas (profile, preferences, etc.)
│   │   └── constants/         # App config, limits, tier definitions
│   └── ui/                    # Shared UI primitives (if needed later)
├── supabase/
│   ├── migrations/            # SQL migrations (tables, RLS, PostGIS, triggers)
│   ├── functions/             # Edge Functions (discover, swipe, etc.)
│   ├── seed.sql               # Dev seed data
│   └── config.toml            # Supabase project config
├── turbo.json                 # Build pipeline config
├── package.json               # Root workspace config
├── .github/
│   └── workflows/             # CI: lint, typecheck, test
└── .env.local                 # Supabase URL + anon key (gitignored)
```

### Dev workflow
- `npx supabase start` -- local Postgres + Auth + Storage + Realtime (Docker)
- `npx expo start` -- Expo dev server for mobile
- `npm run dev --workspace=apps/web` -- Next.js dev server
- `turbo typecheck` -- typecheck all packages in parallel
- `npx supabase db diff` -- generate migrations
- `npx supabase functions serve` -- local Edge Function testing

---

## 10. CI/CD & Deployment

### GitHub Actions

| Trigger | Action |
|---|---|
| PR opened/updated | Lint, typecheck, test (all workspaces via Turbo) |
| Merge to `main` | Deploy landing page to Vercel (auto), push Supabase migrations to staging |
| Manual promote | Push Supabase migrations to production, EAS Build for app stores |

### Deployment targets

| Component | Where | How |
|---|---|---|
| Next.js landing page | Vercel | Auto-deploy on push to `main` |
| Supabase | Supabase Cloud | `npx supabase db push` + `npx supabase functions deploy` |
| Mobile app | App Store / Play Store | EAS Build + EAS Submit |
| OTA updates | Expo | `eas update` for JS-only changes |

### Environments
- **Local**: `npx supabase start` (Docker) + Expo dev server
- **Staging**: Separate Supabase project, Vercel preview deployments
- **Production**: Production Supabase project, Vercel production, app store builds

### Monitoring
- Sentry for error tracking (Expo app + Edge Functions)
- Supabase Dashboard for DB performance, auth metrics, storage usage
- Vercel Analytics for landing page
