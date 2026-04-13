# Sanga -- Dating App

## What is this?
A location-based dating app. Supabase-native, mobile-first MVP.

## Architecture (approved)
Full design spec: `docs/superpowers/specs/2026-04-13-sanga-architecture-design.md`
Product requirements: `constitution tinder file.docx`

## Key Decisions
- **Supabase replaces everything it can**: DB, Auth, Storage, Realtime, Edge Functions, Sessions
- **Expo (React Native)** for mobile (not bare React Native CLI)
- **Next.js** for landing page only (not a web app) -- deployed on Vercel
- **Mobile-first MVP**: Expo app is the product, web is just marketing
- **No custom backend server**: Client SDK + RLS for most operations, Edge Functions for server-side logic
- **Expo Notifications** for push (FCM + APNs) -- the one thing Supabase can't handle
- **RevenueCat** for in-app purchases, Stripe for web payments
- **Turborepo monorepo** with npm workspaces

## Tech Stack
- Mobile: Expo (React Native) + TypeScript
- Web: Next.js + TypeScript + shadcn/ui
- Backend: Supabase (Database/Auth/Storage/Realtime/Edge Functions)
- Push: Expo Notifications
- Payments: RevenueCat (mobile) + Stripe (web)
- CI/CD: GitHub Actions + Vercel (web) + EAS (mobile)
- Monitoring: Sentry

## Monorepo Structure
```
sanga/
├── apps/mobile/       # Expo app
├── apps/web/          # Next.js landing page
├── packages/shared/   # Types, validation, constants
├── supabase/          # Migrations, Edge Functions, seed data
└── turbo.json
```

## Current Status
- [x] Architecture design spec written and approved
- [x] GitHub repo: digiemployeeautomation/sanga
- [x] Vercel linked and connected to GitHub
- [ ] Implementation plan (next step -- use superpowers:writing-plans skill)
- [ ] Project scaffolding (monorepo, Expo, Next.js, Supabase)
- [ ] Database schema & migrations
- [ ] Auth flows
- [ ] Core features (swipe, match, chat)

## Tools Installed
- MCP: Supabase, Expo, Playwright, Chrome DevTools, Sentry, Figma
- Skills: Expo, Supabase, React Native Best Practices, Superpowers, Vercel
- CLIs: supabase (via npx), turbo, expo, vercel, gh

## Commands
- `npx supabase start` -- local dev (needs Docker running)
- `npx expo start` -- mobile dev server
- `npm run dev --workspace=apps/web` -- landing page dev server
- `turbo typecheck` -- typecheck all packages
