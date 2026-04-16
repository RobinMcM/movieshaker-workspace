# MovieShaker Platform — Architecture

## Platform Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     User Facing Platforms                        │
│                                                                  │
│  movieshaker.com    afilminabox.com    reelinvesting.com        │
└──────────┬───────────────────┬──────────────────┬───────────────┘
           │                   │                  │
           ▼                   ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Shared Infrastructure                         │
│                                                                  │
│  auth.rapidmvp.io          SuperTokens core (shared identity)  │
│  models.rapidmvp.io        openrouter-gateway (AI execution)   │
│  media.rapidmvp.io         media-handler (FFmpeg)              │
│  chatbot.rapidmvp.io       chatbot (virtual co-production)     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Service Map

| Service | Repo | Stack | URL |
|---------|------|-------|-----|
| MovieShakerV2 | `movieshakerv2` | Next.js 16 + FastAPI | movieshaker.com |
| Chatbot | `chatbot` | Next.js 16 | chatbot.rapidmvp.io |
| OpenRouter Gateway | `openrouter-gateway` | FastAPI + Python | models.rapidmvp.io |
| Media Handler | `media-handler` | FastAPI + FFmpeg | media.rapidmvp.io |
| aFilmInABox | `afilminabox` | Vite → Next.js (migrating) | afilminabox.com |
| Auth / RapidMVP | `rapidmvp` | Next.js 16 | auth.rapidmvp.io |

---

## MovieShakerV2 Internal Architecture

```
Browser
  │
  ├── movieshaker.com (Next.js web/)
  │     ├── Marketing pages (public)
  │     ├── Auth UI (SuperTokens)
  │     └── Project workspace (authenticated)
  │           ├── scripts, budgeting, scheduling
  │           ├── characters, moodboard, shotlist
  │           ├── visualize, film-in-a-box, festival
  │           └── AI assistant (chatbot embed)
  │
  └── api.movieshaker.com (FastAPI engine/)
        ├── Auth proxy → auth.rapidmvp.io
        ├── CRUD routers (20 feature modules)
        ├── gateway_client.py → models.rapidmvp.io
        ├── media_handler_client.py → media.rapidmvp.io
        ├── PostgreSQL (primary data store)
        └── Valkey (caching)
```

---

## Authentication Flow

All platforms use the same SuperTokens core:

```
User visits movieshaker.com
  → Login request → api.movieshaker.com/auth/*
  → Caddy proxy → auth.rapidmvp.io/auth/*
  → SuperTokens core issues session cookie
  → HTTP-only cookie set for movieshaker.com

Same user visits afilminabox.com
  → SSO redirect to auth.rapidmvp.io
  → Callback validated
  → New first-party session issued for afilminabox.com
  → User is logged in — no second registration required
```

Each app owns its own first-party session.
Cookies are never shared across top-level domains.
Identity is shared — sessions are not.

---

## AI Request Flow

```
Producer in MovieShaker (browser)
  ↓
Next.js web/ (UI action)
  ↓
FastAPI engine/ (business logic, auth check, credit check)
  ↓
gateway_client.py (internal API call)
  ↓
models.rapidmvp.io (openrouter-gateway)
  ↓
┌─────────────────────────────────────┐
│  OpenRouter (text completions)       │  → GPT-4.1, Gemma, Claude, etc.
│  FAL.ai (image/video/audio)          │  → FLUX, Veo, Kling, Wan, etc.
└─────────────────────────────────────┘
  ↓
Gateway returns result + cost
  ↓
Engine deducts credits from user balance
  ↓
Result returned to browser
```

Credit tracking lives in the engine. The gateway is stateless.

---

## Media Processing Flow

```
Producer action (e.g. compile video clips)
  ↓
FastAPI engine/
  ↓
media_handler_client.py
  ↓
media.rapidmvp.io (FFmpeg API)
  ↓
FFmpeg executes in Docker container
  ↓
Output → presigned URL upload → DigitalOcean Spaces
  ↓
Public URL returned to engine → returned to browser
```

---

## Chatbot Flow

```
Producer on a MovieShaker page
  ↓
Embedded chatbot widget (chatbot.rapidmvp.io)
  ↓
Context: current page mode (scripts, budgets, festivals, etc.)
  ↓
POST /api/chat (chatbot Next.js API route)
  ↓
lib/server/gateway-client.js
  ↓
models.rapidmvp.io/api/execute (OpenRouter text)
  ↓
AI response with production-specific system prompt
```

The chatbot calls the gateway directly — not via the MovieShaker engine.
It is an independent service that shares the same gateway.

---

## Data Architecture

```
PostgreSQL (primary)
  ├── Users + profiles (via SuperTokens user_id)
  ├── Projects + members
  ├── Scripts + scenes + characters
  ├── Budgets + scene costs
  ├── Tram lines (scheduling)
  ├── Moodboard items
  ├── Shot lists
  ├── Video history + compiled videos
  ├── Credit balances + transactions
  └── Email records + auth config

DigitalOcean Spaces (file storage)
  ├── Script files (PDF, FDX)
  ├── Moodboard images
  ├── Generated images
  ├── Generated videos
  └── Extracted frames

Valkey (cache)
  ├── Session data
  ├── API response caches
  └── Rate limiting (media-handler)
```

---

## Deployment Topology

```
DigitalOcean Droplet 1 (movieshaker)
  ├── Next.js web container
  ├── FastAPI engine container
  ├── PostgreSQL container
  ├── Valkey container
  ├── SuperTokens container
  └── Caddy reverse proxy

DigitalOcean Droplet 2 (models)
  ├── openrouter-gateway container
  ├── Valkey container (async job state)
  └── Nginx reverse proxy → models.rapidmvp.io

DigitalOcean Droplet 3 (media)
  ├── FFmpeg API container
  ├── media-handler Docker image
  └── Nginx reverse proxy → media.rapidmvp.io

DigitalOcean Droplet 4 (auth)
  ├── SuperTokens core container
  └── Nginx → auth.rapidmvp.io

DigitalOcean Spaces
  └── File storage (all platforms)
```

---

## Credit System Architecture

```
User purchases tokens or subscribes
  ↓
Credits added to user balance (PostgreSQL)
  ↓
User triggers AI feature
  ↓
Engine checks balance (credits.py)
  ↓
Gateway executes and returns usage cost
  ↓
Engine deducts cost from balance
  ↓
Balance returned to UI
```

Free tier users have zero AI credits.
AI features are blocked at the engine level if balance is zero.

---

## ReelInvesting Architecture (Planned)

```
reelinvesting.com (Next.js — TBD)
  ├── Public: information + chatbot (same chatbot infrastructure)
  ├── Registered: project listings and funding requests
  └── Verified: investment marketplace (invite only, KYC required)
        ├── Identity verification service (TBD)
        ├── Online exam system (TBD)
        └── Investment transaction layer (TBD)
```

Auth shared with MovieShaker via auth.rapidmvp.io.
Investment compliance layer is a separate build (Phase 5+).
