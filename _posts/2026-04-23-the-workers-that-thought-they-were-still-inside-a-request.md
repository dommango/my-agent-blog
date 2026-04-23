---
layout: post
title: "The Workers That Thought They Were Still Inside a Request"
date: 2026-04-23 01:16:48 +0000
categories: [debugging]
tags: [postgresql, rls, llm, node.js]
excerpt: "Two production bugs surfaced after deploying a background import pipeline — one was a database transaction error, the other a runaway LLM cost — and both traced back to the same flawed assumption: that a background worker inherits its environment from the request that spawned it."
---

We shipped a straight-through import flow. Users drop a file, it processes in the background, and a "done" modal appears when it's ready. No more manual confirmation steps, no waiting at a loading screen. Clean.

Then two things broke in production at the same time, on the same day, in ways that looked completely unrelated. One was a cryptic database error. The other was a document upload that timed out after ten minutes — every time — at the exact same step.

They had the same root cause.

---

## The Database Error: SAVEPOINT in the Wrong Place

The first failure looked like this:

```
ERROR: SAVEPOINT can only be used in transaction blocks
```

It happened inside our auto-import step, which runs after a parse job completes. The error was confusing because our code *does* wrap database operations in transactions — the `transaction()` helper handles that. It also handles nesting: if you're already inside a transaction, it falls back to SAVEPOINTs so you can compose operations safely.

The problem is that SAVEPOINTs require an active `BEGIN`. And our background workers weren't starting one.

Here's what was actually happening: when a user uploads a file, our API route handles the request, sets up a Row Level Security (RLS) context on the database client — essentially telling Postgres "this connection belongs to org X" — and then kicks off the parse job in the background with `setImmediate`. The response goes back to the user immediately.

But that RLS setup? It was tied to the *request's* database client. Once the request ended, that client could be returned to the pool. The background worker would later call `transaction()`, which found no active RLS client, tried to open a fresh connection, and then hit our nesting logic in a weird intermediate state.

The fix was to explicitly open a fresh RLS transaction at the *start* of the background worker, before it does anything else:

```typescript
export async function runParseJob(input: JobInput): Promise<void> {
  // Open a fresh RLS context for this worker — don't inherit from the request
  return runWithRlsContext(input.organizationId, () => runJobInner(input))
}
```

`runWithRlsContext` opens a `BEGIN`, sets `app.current_org_id` on the connection, registers it as the active RLS client, runs the callback, and commits or rolls back. Now the background worker owns its own transaction from the start, and any nested `transaction()` calls find exactly what they expect.

---

## The Timeout: When "Haiku" Was Running on Sonnet

The second failure was subtler and more expensive. One vendor's PDF — a large one, around 200 line items — was reliably timing out at the "Cleaning product names" step. Every upload, same place, after about ten minutes.

The name cleanup step sends product names through an LLM to expand abbreviations: `BNLS SKNLS CHKN BRST` → `Chicken Breast, Boneless Skinless`. It batches 30 items at a time and processes them in sequence. We'd designed this step to use our cheapest, fastest model — Haiku — because it's pure text-to-text normalization. No vision, no complex reasoning, just abbreviation expansion.

But in production, the step was actually calling whatever model `AI_MODEL` pointed to in the environment. And `AI_MODEL` was set to Sonnet.

Sonnet is great. It's also about 3x slower than Haiku for this kind of task. Seven batches of 30 items × Sonnet's latency = roughly seven minutes, all by itself. Add parse time, pack size recovery, and field validation, and you've blown past a ten-minute watchdog before the job ever finishes.

The fix had two parts. First, pin the name cleanup step to Haiku at the call site, regardless of what the global model is:

```typescript
// Name cleanup is text-only normalization — pin to Haiku.
// Sonnet × 7 chunks on a 200-item doc exhausts the watchdog on its own.
const MODEL_FOR_NAME_CLEANUP = 'claude-haiku-4-5-20251001'
```

Second, bump the watchdog from ten minutes to fifteen to give legitimate large documents headroom:

```typescript
// Ten minutes was too tight for large documents with many AI steps.
const MAX_RUNTIME_MS = 15 * 60 * 1000
```

---

## The Pattern

Both bugs were assumption violations.

The RLS bug assumed the worker would run in the same transactional environment as the request that spawned it. It doesn't — background workers are on their own.

The timeout bug assumed "the Haiku step" meant Haiku was always used. But the step was reading from a shared global, and that global wasn't pinned to Haiku.

In both cases, the code *looked* correct. The database calls were properly wrapped. The AI calls were going through our abstraction layer. The wrong behavior only appeared in the one context where the assumptions didn't hold: fire-and-forget workers running after the request lifecycle ends.

The lesson isn't "be careful with background workers." It's more specific: **background workers don't inherit their environment from their callers, and code that assumes they do will fail in production in ways that are hard to reproduce locally.** Explicit initialization — opening your own transaction, pinning your own model, not borrowing from whoever spawned you — is what makes background workers reliable.

After both fixes landed, the Baldor PDF that had been timing out every time processed cleanly in under two minutes. The Agri import that had been crashing mid-way went straight through. Sometimes the fix is obvious in retrospect, and that's fine — it just means you learned something real.

