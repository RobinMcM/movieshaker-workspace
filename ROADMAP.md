# MovieShaker Platform — Roadmap

## Current State (April 2026)

MovieShakerV2 is live in production with real users.
Core production management features are complete and working.
AI integration is partially built and being expanded.

### What is complete
- ✅ Project management (CRUD, members, administration)
- ✅ Script upload, parsing, and scene breakdown
- ✅ Budgeting and scene cost management
- ✅ Shoot day scheduling (tram lines)
- ✅ Character and cast management
- ✅ Moodboard with AI image generation
- ✅ Shot list management
- ✅ Scene visualization (AI image-to-video)
- ✅ Film in a Box (AI script/film generation)
- ✅ Film Festival analysis (AI-powered)
- ✅ User profile and credit system
- ✅ Admin panel (users, email, auth config)
- ✅ Email delivery (Resend + webhook)
- ✅ SuperTokens authentication
- ✅ DigitalOcean Spaces file storage
- ✅ openrouter-gateway (text + FAL media)
- ✅ media-handler (FFmpeg API)
- ✅ Chatbot (MovieShaker modes, gateway connected)
- ✅ AI assistant endpoint (engine/ai_assistant.py)

### What is in progress
- 🔄 AI assistant UI integration (frontend not yet built)
- 🔄 Free tier enforcement (film count + eighths limits)
- 🔄 Token purchase flow (Stripe integration pending)
- 🔄 Subscription management (pending)

---

## Phase 1 — AI Assistant Integration
**Goal:** Embed the chatbot into MovieShaker pages with page context.

- [ ] Embed chatbot widget into MovieShakerV2 web
- [ ] Pass page context (current section, project data) to chatbot
- [ ] Connect chatbot to engine `/api/ai/chat`
- [ ] Add AI assistant admin configuration page
- [ ] Test end-to-end: producer asks question → gets context-aware answer

**Services:** `movieshakerv2`, `chatbot`
**Complexity:** Low — infrastructure already built

---

## Phase 2 — Free Tier Enforcement + Monetisation
**Goal:** Enforce free tier limits and enable AI purchases.

- [ ] Enforce 3-film limit per user (engine)
- [ ] Enforce 120-eighths limit per film (engine, scripts.py)
- [ ] Block AI features for zero-credit users (engine, credits.py)
- [ ] Stripe integration for token purchase
- [ ] Stripe integration for AI subscription (monthly/annual)
- [ ] Credit top-up flow (web UI + engine)
- [ ] Subscription status management
- [ ] User-facing credit balance display
- [ ] Upgrade prompts when AI credit runs out

**Services:** `movieshakerv2`
**Complexity:** Medium — requires Stripe, subscription state management

---

## Phase 3 — Media Handler Expansion
**Goal:** Complete FFmpeg function coverage via the API.

Missing operations to implement (one route each):
- [ ] `probe` — video metadata (duration, codec, resolution, fps)
- [ ] `audio-extract` — extract audio track as mp3/aac
- [ ] `audio-replace` — replace audio track on a video
- [ ] `speed` — change playback speed (fast/slow motion)
- [ ] `reverse` — reverse video playback
- [ ] `fade` — fade in/out transitions
- [ ] `subtitle` — burn in subtitles/captions (SRT/ASS)
- [ ] `split` — split video at timestamp
- [ ] `gif` — convert clip to animated GIF
- [ ] `thumbnail-grid` — extract multiple frames as contact sheet

Each follows the same pattern: `schemas.py` → `ffmpeg.py` → `main.py`.

**Services:** `media-handler`
**Complexity:** Low per endpoint — pattern is well established

---

## Phase 4 — aFilmInABox Next.js Migration
**Goal:** Fix SuperTokens auth and align with platform stack.

- [ ] Next.js App Router scaffold
- [ ] SuperTokens auth (same pattern as MovieShakerV2)
- [ ] WebSocket server compatible with Next.js
- [ ] Port camera management components from Vite
- [ ] QR code generation in Next.js API route
- [ ] WebRTC signalling via Next.js WebSocket
- [ ] Valkey session state
- [ ] Subscription/registration gate for editing functions
- [ ] media-handler integration for video processing
- [ ] Docker deployment updated
- [ ] Decommission Vite stack

**Services:** `afilminabox`
**Complexity:** Medium — architecture is well understood, migration work

---

## Phase 5 — ReelInvesting Phase 1 (Information + Chatbot)
**Goal:** Launch reelinvesting.com as an information and guidance platform.

- [ ] Next.js scaffold for reelinvesting.com
- [ ] SuperTokens auth (same shared identity)
- [ ] Information pages about film funding
- [ ] Chatbot integration (same chatbot infrastructure)
  - New mode: `funding` — funding guidance system prompt
  - New mode: `marketing` — film marketing guidance
- [ ] Register reelinvesting.com with auth.rapidmvp.io
- [ ] Domain and Caddy configuration

**Services:** `chatbot` (new modes), new `reelinvesting` repo
**Complexity:** Low — reuses existing infrastructure

---

## Phase 6 — ReelInvesting Phase 2 (Investment Marketplace)
**Goal:** Launch invite-only investment and services marketplace.

- [ ] Project listing pages for producers
- [ ] Investor/advisor application flow
- [ ] Online exam system (investment risk + production value)
- [ ] Identity verification (KYC) integration
- [ ] Invitation management (admin-controlled)
- [ ] Offer listings: financial investment, services, advice
- [ ] Matching system: producer projects ↔ investor offers
- [ ] Communication layer between producers and investors
- [ ] Legal compliance review (financial promotion rules)
- [ ] Transaction facilitation layer (TBD — regulatory dependent)

**Services:** new `reelinvesting` repo
**Complexity:** High — regulated environment, compliance requirements,
identity verification, legal review required before launch

---

## Phase 7 — Platform Maturity
**Goal:** Production hardening, analytics, and scale.

- [ ] Usage analytics (AI feature adoption, credit burn rates)
- [ ] Admin analytics dashboard
- [ ] Automated credit alerts (low balance notifications)
- [ ] Subscription management portal (upgrade, downgrade, cancel)
- [ ] Platform-wide notification system
- [ ] API rate limiting per user tier
- [ ] Performance optimisation (caching strategy review)
- [ ] Mobile-first UI audit across all platforms
- [ ] Accessibility audit

---

## Build Principles

These principles apply to every phase:

1. **Small deliberate steps** — one task, one file, one confirmation
2. **Test before deploy** — canary/smoke test every service change
3. **No autonomous agents** — every change reviewed before applied
4. **Document as you build** — CLAUDE.md and README updated with each phase
5. **Gateway is stateless** — cost tracking always lives in the engine
6. **Auth is always first-party** — never share cookies across domains
7. **Free tier first** — enforce limits before building premium features

---

## Immediate Next Steps (from today)

1. Complete AI assistant UI (Phase 1)
2. Design and implement free tier limits (Phase 2 start)
3. Choose and integrate Stripe (Phase 2)
4. Begin media-handler `probe` endpoint (Phase 3 — low risk, high value)
