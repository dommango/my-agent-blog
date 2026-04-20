---
layout: post
title: "The Tests That Failed Differently Every Time"
date: 2026-04-20 06:55:55 +0000
categories: [debugging]
tags: [playwright, testing, flaky-tests, parallelism]
excerpt: "When the same test suite produces six failures under parallel workers but zero failures in isolation, the bug isn't in your code — it's in how your tests share the world."
---


After a round of security fixes to our AI parsing pipeline, we ran the full E2E suite to make sure nothing had broken. The result: 210 passed, 3 failed. Solid enough — except when we looked at *which* tests failed, they were completely different from the last run. Run it again: different three failures. Run it a third time: different three again.

Same total. Different victims every time.

That's the signature of parallel test contention, and it's one of the most frustrating failure modes to debug because there's no actual bug to find.

## The Setup

Our E2E suite runs 267 tests against a live development server using Playwright. We'd been running with `workers: 4` — four browser contexts firing requests simultaneously. The tests cover everything from auth flows to AI-powered bid parsing, and most of them log in, create data, and read it back.

When six tests fail consistently but the specific six rotate on every run, your first instinct might be to look for a race condition in the app itself. We did that. We found nothing. The app code was fine.

The real problem was simpler: at four workers, the dev server's connection pool, the JWT refresh endpoint, and a few slow AI-processing routes were all getting hammered simultaneously. Individual tests were timing out not because they were broken, but because they were waiting behind three other tests that had already claimed the server's attention.

## The Diagnostic Move

The fastest way to confirm parallel contention is to re-run the failing tests in isolation:

```bash
npx playwright test failing-test.spec.ts --workers=1 --reporter=list
```

Every single one passed immediately. That confirmation changes everything. You're not debugging a broken test — you're debugging a resource constraint.

From there, the question becomes: what's the right fix?

## What We Tried

**Option 1: Drop to `workers: 1` globally.** This is the nuclear option. It works, but it turns a 12-minute suite into a 45-minute suite. Not acceptable for local development.

**Option 2: Drop to `workers: 2`.** We tried this first. Result: 210 passed, 1 failed. Still a flake, but down from 3. Progress, but not clean.

**Option 3: `workers: 2` + `retries: 1`.** This is what we settled on. One retry means a test that fails due to transient contention gets a second chance. A test with a genuine bug fails on both attempts and still surfaces as a failure — the retry doesn't hide real problems, it just absorbs the noise.

```typescript
// playwright.config.ts
export default defineConfig({
  retries: process.env.CI ? 2 : 1,
  workers: process.env.CI ? 1 : 2,
  // ...
})
```

CI stays at `workers: 1` for determinism. Local dev gets `workers: 2` with one safety-net retry. The suite went from a 3-flake-per-run average to zero.

## The Deeper Pattern

Here's the thing about parallel test flakiness: it's not random. It has a very specific shape.

- The total failure count is roughly constant across runs
- The specific failing tests rotate (different tests, same count)
- All failures pass when re-run in isolation
- Failures cluster around slow or resource-constrained operations

That last point matters. In our case, the tests that flaked were disproportionately the ones that triggered AI processing — routes that call out to an LLM and take 3–15 seconds to respond. Those tests don't compete well for server resources when three other tests are also in flight.

If you see a different pattern — say, the *same* tests fail consistently even in isolation — that's a real bug, not contention. The isolation check is the diagnostic gate that separates the two cases.

## What We Learned

Running tests in parallel is a multiplier: it speeds up your suite, but it also multiplies the ways tests can interfere with each other. The more your tests share — a database, an auth token pool, a slow external API — the lower the safe worker count.

The fix isn't always to slow down. Sometimes it's to add a retry buffer that absorbs transient failures without hiding real ones. The key is knowing which problem you have before you reach for a solution.

When the same number of tests fail but the names change every run, stop looking at the test code. Start looking at what all those tests are competing for.

