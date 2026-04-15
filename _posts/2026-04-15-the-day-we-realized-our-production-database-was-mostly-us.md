---
layout: post
title: "The Day We Realized Our Production Database Was Mostly Us"
date: 2026-04-15 02:29:22 +0000
categories: [til]
tags: [postgresql, database, testing, saas]
excerpt: "When we queried our production database for the first time after real users signed up, we found seven organizations — five of which were our own test runs in disguise."
---

We had real users. We were excited. Then we queried the production database.

```sql
SELECT name, plan, created_at FROM organizations ORDER BY created_at;
```

Seven rows came back. Two were the actual signups from that day. The other five were us — a demo seed account, three E2E test runs from the past few weeks, and an abbreviation-parsing experiment we'd spun up to validate a feature. All sitting quietly in production, indistinguishable from real customers unless you knew the naming patterns by heart.

## How It Happens

When you're building a SaaS, you create accounts constantly: seeding demo data, running end-to-end tests, trying something manually that should have been in a test fixture. The problem is that each of those operations leaves an artifact. During development, that's fine — you're pointing at a local database, and `docker compose down -v` nukes everything.

But somewhere between "local testing" and "shipped," there's a moment when your E2E suite starts running against production because that's where the real environment lives. Your demo seed script runs once to make the welcome screen look nice. You sign up manually to try the onboarding flow as a fresh user. Each of these creates an org, and none of them clean up after themselves.

By the time your first real users show up, the database already has a history. Most of it is you.

## The Signal We Used

Figuring out what to delete was the interesting part. We had two reliable signals:

**Creation timestamp.** Our real users signed up that day. Every test org was weeks old. This alone was nearly sufficient, but timestamps can mislead — a stale demo account matters too.

**Naming conventions.** Our E2E suite had created orgs with names like "E2E Test User" and "E2E Polish Test." Hard to misread. The abbreviation experiment was less obvious, but still clearly not a restaurant operator's chosen org name.

Together, these two signals let us partition the table cleanly: anything with a test-looking name or a creation date predating real signups was a candidate for deletion.

## The Delete

We already had a cascade `DELETE` wired into our admin API for exactly this kind of maintenance:

```
DELETE /admin/orgs/:orgId
```

The endpoint used `SET LOCAL ROLE postgres` to bypass Row Level Security, then deleted in foreign key order — bid items, products, vendors, members, then the org itself. Running it five times took about thirty seconds.

The database went from seven organizations to two. Both of them are people we don't know personally. That felt right.

## What We'd Do Differently

The real fix is upstream: E2E tests should create orgs with a known prefix (`e2e-`) and a cleanup hook that runs at the end of every test suite. Demo seeds should target a dedicated environment. Manual testing accounts should live in staging, not production.

None of that is complicated. It's just the kind of discipline that doesn't feel urgent until you're sitting in front of a production database on launch day, trying to remember which orgs are real.

The good news: once you've done the cleanup once, you remember to build the guardrails.
