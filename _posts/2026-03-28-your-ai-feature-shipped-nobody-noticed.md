---
layout: post
title: "Your AI Feature Shipped. Nobody Noticed."
date: 2026-03-28 21:25:53 +0000
categories: [integration]
tags: [ai, ux, react, product-design]
excerpt: "We shipped AI-powered parsing and semantic search — then realized users had no idea it was running, so we built a status badge and changelog page to make the invisible visible."
---


We spent weeks building an AI-enhanced parsing pipeline. Claude Haiku normalizing abbreviated vendor names. pgvector embeddings replacing fuzzy string matching with semantic similarity. Confidence scores surfaced in the UI. It worked great.

Then we showed it to users. They had no idea anything had changed.

That's the quiet trap of backend AI: it improves things silently. No error message disappears, no new button appears — the app just works a bit better. For a demo or investor walkthrough, "trust me, it's smarter now" is a hard sell.

## The Problem with Invisible Intelligence

Our app helps restaurant operators compare vendor bids and make purchasing decisions. The AI improvements lived entirely in the import pipeline: smarter name extraction, fewer mismatched products, better category guessing. All invisible.

We had three flavors of "AI is doing something":
- **Active and confident** — AI enhanced this import automatically
- **Active but uncertain** — AI made a suggestion, human confirmed it
- **Disabled** — no API key configured, running on deterministic logic only

Users couldn't tell which mode they were in. Owners couldn't toggle it. Demo viewers didn't know to look for it.

## What We Built

Two things: a header badge and a changelog page.

**The badge** lives next to the existing health indicator in the top nav. A colored dot (green = automatic, yellow = manual, gray = disabled) with the label "AI". Click it and a popover opens:

```
● AI Active (automatic)
─────────────────────────
AI Parsing     claude-haiku ✓
Embeddings     847 / 847 products indexed

[ automatic ]  [ manual ]  [ disabled ]
→ What's New
```

Owners and managers can toggle the mode inline. Viewers see the status but can't change it. The popover links to the changelog.

**The changelog page** (`/whats-new`) is a simple static page listing the three release milestones with plain-language descriptions of what changed. Not a technical changelog — more like a product release note. "AI now reads abbreviations like BNLS SKNLS CHKN and understands they mean Chicken Breast Boneless Skinless." That kind of thing.

## The Interesting Engineering Bit

The badge polls a single `GET /settings/features` endpoint that returns the full picture in one shot:

```typescript
{
  ai: {
    enabled: boolean,
    mode: 'automatic' | 'manual' | 'disabled',
    model: string,
  },
  embeddings: {
    totalProducts: number,
    embeddedProducts: number,
    coveragePercent: number,
  }
}
```

The backend resolves this with three parallel queries — AI config from env, mode setting from the org row, embedding counts from the products table. No sequential waterfalls, no N+1. The frontend polls every 60 seconds with a 30-second stale time, so it stays fresh without hammering the server.

The mode toggle calls `PATCH /settings/features/ai-mode`, validated with Zod, gated to manager/owner roles. Role checking happens at the middleware layer, not scattered through the handler.

## What We Learned

**AI features need a window.** Even if the AI is doing exactly what it should, users need to see that it's running. A status badge serves double duty: it's a trust signal ("yes, this is the AI-enhanced version") and a diagnostic tool ("oh, it's disabled because the API key isn't set").

**The mode toggle matters for demos.** Being able to switch between automatic/manual/disabled live in the UI lets you show the difference. Turn AI off, import a messy bid sheet, watch it struggle. Turn it back on, import the same sheet, show the improvement. That's a much more compelling demo than "it's better, trust me."

**Plain language beats feature names.** The changelog doesn't say "pgvector cosine similarity for semantic product matching." It says "products with different abbreviations now match correctly across vendors." Same thing, different audience.

The invisible AI is now visible. Turns out that was the last missing feature.

