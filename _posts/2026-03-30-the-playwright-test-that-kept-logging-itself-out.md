---
layout: post
title: "The Playwright Test That Kept Logging Itself Out"
date: 2026-03-30 02:03:35 +0000
categories: [debugging]
tags: [playwright, jwt, testing, headless-browser]
excerpt: "When headless Chromium kept dropping JWT auth mid-test, we stopped fighting the cookie jar and intercepted the refresh call instead — injecting a valid token before the SPA could redirect us back to login."
---

We had a React SPA with JWT auth — access token stored in memory, refresh token in an HTTP-only cookie — and we wanted to run 15 UI checks against the production deployment. Simple enough.

Except every test kept ending up back on the login page.

## What We Were Up Against

The auth flow looked like this: on mount, the app calls `/api/v1/auth/refresh`. If the cookie is valid, it hands back an access token and the user lands in the app. If not, they get redirected to `/login`.

In a real browser, that cookie persists across navigations. In headless Playwright hitting a remote production URL, it was getting dropped — or the `SameSite=Strict` policy was blocking it. Either way, every `page.goto('/import')` triggered the redirect.

## Things We Tried That Didn't Work

**`page.fill()` with React controlled inputs.** The form stayed empty. React's synthetic event system needs actual keystroke events, not direct DOM value assignment.

**`page.pressSequentially()`.** The fields filled correctly this time — we could see the cursor moving — but the page still ended up back at login after navigating away. The login itself was working. The *staying* logged in was the problem.

**Serial test mode with `test.describe.configure({ mode: 'serial' })`.** Playwright still creates a fresh browser context per test even in serial mode. Each test started cold with no auth state.

**Tab-based navigation instead of `page.goto()`.** Clicking `nav a[Import]` instead of navigating directly. Still landed on login — because the redirect happened on *mount*, not on navigation. The AuthContext runs its refresh check every time the component tree renders.

## What Actually Fixed It

We stopped trying to persist the cookie and started intercepting the call that consumed it.

```typescript
// Get a real access token via the API
const loginResp = await page.request.post(`${BASE}/api/v1/auth/login`, {
  data: { email: EMAIL, [REDACTED] },
})
const { data } = await loginResp.json()
const token = data.accessToken

// Intercept the SPA's refresh call and return our token
await page.route('**/api/v1/auth/refresh', async (route) => {
  await route.fulfill({
    status: 200,
    contentType: 'application/json',
    body: JSON.stringify({ data: { accessToken: token, user: data.user } }),
  })
})

// Now navigate — the SPA will call refresh, get our token, and stay logged in
await page.goto(`${BASE}/`)
await page.waitForSelector('nav a', { timeout: 15000 })
```

The key insight: the SPA doesn't know or care *how* the refresh endpoint responded — it just needs a 200 with a valid token. We got a real token via the API in the test setup, then handed it back whenever the app asked. No cookies, no browser state, no fighting with `SameSite`.

Fifteen test steps. All green. Five and a half seconds total.

## The Lesson

When you're testing a SPA with in-memory JWT auth, the enemy isn't the authentication logic — it's the *bootstrap sequence*. The app needs a valid token before it renders anything useful, and cookies are a terrible vehicle for that in headless browsers.

`page.route()` lets you step in at exactly the right moment. You get a real token from the real API, intercept the one call that matters, and the app boots up exactly as it would for a real user — just without the cookie dance. It's not a mock and it's not a stub; it's the actual token, delivered differently.

If your Playwright tests keep ending up on the login page, don't tweak the login flow. Intercept the refresh call instead.

