---
layout: post
title: "The 500 That Was Poisoning Every Page"
date: 2026-04-20 02:24:38 +0000
categories: [debugging]
tags: [postgresql, playwright, e2e-testing, debugging]
excerpt: "We ran our E2E suite and found three bugs working together to tank 80% of our tests — a bad SQL CTE, a missing database grant, and a duplicate button label that Playwright's strict mode refused to tolerate."
---

We ran our full Playwright suite before a deploy and landed at 99 passed out of 250 tests. That's not a test problem — that's a systemic failure hiding behind test output.

Three bugs were operating in tandem. Each looked unrelated. Together they were responsible for the majority of the failures.

## Bug 1: The CTE With a Trailing Comma

Our bid comparison query was built as a chain of Common Table Expressions — a series of named subqueries that pass data down to a final `SELECT`. Somewhere in a refactor, a comma snuck in after the last CTE definition:

```sql
WITH ranked_bids AS (
  SELECT ...
),
price_stats AS (
  SELECT ...
),  -- ← this comma has nowhere to go
SELECT * FROM price_stats ...
```

PostgreSQL is unforgiving about this. That trailing comma makes the parser expect another CTE block before the final `SELECT`. The result: a `syntax error at or near "SELECT"` on every request to that endpoint.

The fix was a one-character deletion. But because this endpoint was called on page load across multiple views, the error was appearing in 17 different test failures — all of them looking like navigation or rendering problems, not a SQL issue.

## Bug 2: The Grant That Never Ran

A migration added a `users` table lookup to support a members management view. The migration created the table and set up the right foreign keys — but never granted the application database role permission to read from it.

In development, the DB connection often runs as a superuser, so this works fine locally. In the test environment and production, the connection uses a least-privilege role. The result: `permission denied for table users` on every request that touched org membership data.

The fix was a small migration — a single `GRANT SELECT ON users TO app_tenant` — but finding it meant correlating "Members tab returns 500" with "this table was added in migration 039 but the grant was never included."

## Bug 3: The Duplicate That Playwright Refused to Guess About

Playwright's strict mode means that `getByText('Upload Bid Sheet')` will throw an error if that text appears more than once on the page — even if one match is a `<button>` and the other is a `<p>`. It won't pick one. It just stops.

Our dashboard had exactly this situation: a top-level call-to-action button labeled "Upload Bid Sheet" and, lower on the page in a quick-actions grid, a card with a paragraph reading "Upload Bid Sheet." Two different elements, same text, same page.

The fix was renaming the card's label to something distinct ("Import Bid"). The button kept its label. Playwright's strict mode was no longer ambiguous about which element we meant.

## What the Pattern Looks Like

When most of your E2E failures are timeouts and navigation errors — not assertion failures — the problem is usually upstream. Timeouts mean pages didn't load. Pages don't load because APIs failed. APIs fail silently.

The way we found these was simple: after each test run, we isolated the failing tests and looked at what they had in common before the actual assertion step. Tests across completely different features were all hitting the same two endpoints. That's the tell.

## The Cumulative Result

After patching all three:

- Pass count went from **99 → 114 → 145** across two fix rounds
- Both backend endpoints returned 200 on smoke curl
- Zero occurrences of the original error strings in the new test logs
- The remaining failures were all pre-existing flaky specs against AI and PDF parsing — unrelated to these bugs

The lesson isn't subtle: when a lot of tests fail at once, look for a small number of shared upstream failures before you start reading individual test output. A single bad SQL query can make a hundred unrelated tests look broken.

