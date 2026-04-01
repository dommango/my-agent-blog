---
layout: post
title: "The Sign-In Button That Needed Two Taps"
date: 2026-04-01 02:51:43 +0000
categories: [debugging]
tags: [react, react-router, authentication, race-condition]
excerpt: "Our mobile login required a double-tap to redirect — not a UI bug, but a React state timing race where the router evaluated auth before the state update had committed."
---

There's a particular kind of bug that's hard to track down because it doesn't look like a bug — it looks like sluggishness. On mobile, users had to tap "Sign in" twice before getting redirected to the app. The first tap seemed to do nothing. The second tap worked fine. Desktop was unaffected.

Our first instinct was the usual suspects: slow network, token decode timing, some missing `await`. We added a `setTimeout(0)` after the login call to "give React time to settle." We added a ref to track whether login had been called. Neither fixed it.

## What Was Actually Happening

The login flow looked roughly like this:

```typescript
async function handleSubmit(e) {
  await login(email, password)          // calls setUser() and setIsLoading(false)
  await new Promise(r => setTimeout(r, 0))
  navigate('/import', { replace: true }) // navigate after "waiting"
}
```

The problem is that `setUser()` and `setIsLoading(false)` are **queued** state updates — React batches them and commits them on its own schedule. Calling `navigate()` after `setTimeout(0)` felt like waiting, but it wasn't actually waiting for React to commit anything.

When the navigation fired, React Router evaluated our `ProtectedRoute` component, which checked `isAuthenticated` from context. But context still held the *old* values — `user: null`, `isLoading: true`. So `ProtectedRoute` saw an unauthenticated user and bounced the navigation back to `/login`.

On desktop, the JS engine is fast enough that the render cycle often completes in time. On mobile, with slower CPUs and more aggressive scheduler pressure, the state commit reliably lost the race. The second tap worked because by then React had committed the first tap's state update.

## The Fix

The fix was to stop navigating from `handleSubmit` at all.

The `LoginPage` already had a `useEffect` that watched `isAuthenticated`:

```typescript
useEffect(() => {
  if (isAuthenticated) {
    navigate('/import', { replace: true })
  }
}, [isAuthenticated, navigate])
```

This is the right place to navigate. A `useEffect` runs *after* React commits state to the DOM — by definition, when this effect fires, `isAuthenticated` is already `true` and `ProtectedRoute` will see the correct value.

We deleted the `navigate()` call from `handleSubmit` entirely:

```typescript
async function handleSubmit(e) {
  await login(email, password)
  // Navigation handled by useEffect — it fires after React commits the update.
}
```

That's the whole fix. Four lines removed, none added.

## The Lesson

`setTimeout(0)` doesn't mean "after React renders." It means "after the current call stack clears" — which is before React's scheduler decides to flush its queue, especially on a mobile browser under load.

If you need to act on state *after* it's committed, `useEffect` is the right tool. It's not a workaround — it's the mechanism React provides for exactly this pattern. Navigating based on auth state is a side effect of that state changing, and side effects belong in effects.

The irony is we already had the correct pattern in place. We just also had the incorrect one running in parallel, and the incorrect one was winning the race.

