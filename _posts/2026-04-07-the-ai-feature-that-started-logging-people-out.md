---
layout: post
title: "The AI Feature That Started Logging People Out"
date: 2026-04-07 15:34:50 +0000
categories: [debugging]
tags: [jwt, authentication, ai-integration, typescript]
excerpt: "Adding AI to a parsing pipeline changed how long user operations take — and our 15-minute JWT tokens didn't get the memo."
---

We shipped AI-powered bid parsing — Haiku reads vendor invoices, expands abbreviations, normalizes product names, recovers missing SKUs. It works well. Users liked it.

Then we started getting bug reports: people were getting kicked out mid-import, right when the AI step was running.

## What Was Happening

The app uses short-lived JWT access tokens for authentication — 15 minutes by default. That's a pretty standard choice. Tokens expire quickly, reducing the blast radius if one gets intercepted.

The problem: our AI parsing operations take 3–30 seconds. Vendor detection, structure analysis, line parsing, sanity checking, name normalization — that's six separate Haiku calls, some batched, but still real latency.

If a user started an import at minute 14:45 of their session, the access token expired while the AI was still running. The next API call came back 401. The import silently failed. The user saw a login redirect and lost their work.

It wasn't a new bug — it was a new assumption violation. Before AI, import operations took under a second. The 15-minute window was effectively infinite. Now it wasn't.

## Two Things We Fixed

**First: the TTL.** We extended the access token from 15 minutes to 60. Not forever — you still want short-lived tokens — but 60 minutes is a reasonable session window for a professional tool. Refresh tokens (7 days) handle re-authentication when you actually walk away. The 15-minute default was borrowed from API docs without thinking about whether it fit a workflow tool.

**Second: proactive refresh.** Even 60 minutes isn't enough on its own. If someone leaves a tab open, their token will eventually expire. We added a `refreshIfExpiringSoon()` helper — called before every AI operation — that silently exchanges the token for a fresh one if it's within 2 minutes of expiry:

```typescript
async function refreshIfExpiringSoon(): Promise<void> {
  const token = getAccessToken()
  if (!token) return
  const payload = decodeToken(token)
  const expiresIn = payload.exp * 1000 - Date.now()
  if (expiresIn < 2 * 60 * 1000) {
    await refreshToken() // silent, no redirect
  }
}

// Called before every AI parse operation
await refreshIfExpiringSoon()
const result = await runAIParsePipeline(lines)
```

No user-visible behavior. No interruption. The token stays fresh through whatever the AI needs to do.

## The Broader Lesson

When you add AI to an existing system, you're not just changing outputs — you're changing operation duration. A flow that used to resolve in 200ms now takes 15 seconds. That ripples into things you never thought about: session management, loading states, timeout thresholds, mobile keep-alive behavior.

The fix here was small. The insight is larger: any time you add meaningful latency to a user operation, go back and audit the assumptions that were built around "this is fast." Some of them won't hold anymore.

