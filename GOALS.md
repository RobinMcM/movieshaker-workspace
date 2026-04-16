# MovieShaker Platform — Goals

## Vision

A complete ecosystem for indie film makers — from first script page
to finished film, funded, and distributed.

Three interconnected platforms sharing a single identity:

```
movieshaker.com     Film production management
afilminabox.com     Multi-camera capture and management
reelinvesting.com   Film funding and investment marketplace
```

One registration gives access to all three.

---

## MovieShaker.com

### Purpose
The core production platform. Every tool a production team needs
from script to final film, in one place.

### Free Tier
- Up to 3 films (each up to 15 minutes / 15 pages / 120 eighths)
- Full access to: scripts, budgeting, scheduling, characters,
  moodboard, shot lists, crew, objects, project administration
- No AI features included

### Paid Tier — AI Access
AI features are commercial. Access via:
- **Subscription** — monthly/annual recurring access to AI
- **Token purchase** — pay-as-you-go AI credits

AI features that require purchase:
- Moodboard image generation
- Scene visualization (image-to-video)
- Film in a Box (AI script/film generation)
- Film Festival analysis
- AI Assistant chat support

### Commercial Goal
Convert free users to AI subscribers or token purchasers.
The free tier provides enough value to build a production team
workflow. AI accelerates that workflow — that is the value
proposition for the paid tier.

---

## aFilmInABox.com

### Purpose
Multi-camera capture platform for film sets.
Connect iPhone cameras via QR code, stream live, manage recordings.

### Free Tier
Free forever — basic camera connection and recording.

### Registered Tier
Registration unlocks editing and file management functions.
No payment required to register.

### Paid Tier
Full editing and media management via subscription.
Editing uses `media-handler` (FFmpeg) under the hood.

### Current Status
⚠️ **Migration required** — SuperTokens auth integration has known
issues with the current Vite stack. Migration to Next.js is the
next build priority for this platform.

---

## ReelInvesting.com

### Purpose
Film funding and investment marketplace.
Helps producers fund and market their films.
Owned by MovieShaker.

### Phase 1 — Information Site
- Static information about film funding and investment
- AI-powered chatbot (same chatbot infrastructure as MovieShaker)
- Guides producers on funding routes, grants, and marketing strategies

### Phase 2 — Investment Marketplace
A curated, invite-only marketplace where:
- Investors can offer financial investment in productions
- Industry professionals can offer free services and advice
- Producers can present their projects for backing

### Compliance Requirements (Phase 2)
Every participant in the investment marketplace must:
1. Complete an online exam (understanding investment risks and production value)
2. Provide proof of identity (KYC)
3. Receive and accept an invitation

This is a regulated environment. Participants are not anonymous.
The platform is not open to the public — invite only.

### Commercial Goal
ReelInvesting connects investment capital with film productions.
The platform earns through transaction facilitation, not advertising.

---

## Shared Infrastructure Goals

### Single Identity
One SuperTokens registration at `auth.rapidmvp.io` provides access
to MovieShaker, aFilmInABox, and ReelInvesting automatically.
The user never registers twice.

### AI Infrastructure
All AI features across all platforms route through `openrouter-gateway`
at `models.rapidmvp.io`. One gateway, multiple platforms, unified
cost tracking and model management.

### Measurement Standard
Film length across the platform is measured in **eighths**:
- 1 page = 8 eighths = approximately 1 minute of screen time
- 15 minutes = 15 pages = 120 eighths
- Free tier limit: 3 films × 120 eighths = 360 eighths total

---

## Summary

| Platform | Model | Revenue |
|----------|-------|---------|
| MovieShaker | Freemium | AI subscriptions + token sales |
| aFilmInABox | Freemium | Editing subscriptions |
| ReelInvesting | Invite-only | Investment facilitation |
| Shared auth | Infrastructure | — |
