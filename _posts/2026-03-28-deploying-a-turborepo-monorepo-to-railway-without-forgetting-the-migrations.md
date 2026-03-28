---
layout: post
title: "Deploying a Turborepo Monorepo to Railway (Without Forgetting the Migrations)"
date: 2026-03-28 15:22:45 +0000
categories: [tooling]
tags: [railway, turborepo, postgresql, deployment]
excerpt: "Deploying a Turborepo monorepo to Railway is straightforward once you understand how to scope build commands per service and fold database migrations safely into the start command."
---

We had a working app — production-ready, all eight development phases complete, 440 tests green — and zero deployment infrastructure. Classic. The time had come to actually ship the thing to Railway, and what felt like it should be a 20-minute task turned into a useful education on how Railway thinks about monorepos.

## The Setup

Our app is a Turborepo monorepo with pnpm workspaces: an Express API, a React SPA, a handful of shared packages, and a PostgreSQL database sitting at 30 migrations. Nothing exotic, but also not a single `package.json` pointing at one `index.js`. Railway's default assumptions — "detect framework, build, deploy" — needed a little guidance.

## How Railway Sees a Monorepo

Railway treats each deployable unit as a separate **service**. That's actually the right mental model: the API and the frontend are different things that scale independently, deploy on different triggers, and have different environment variables. You set a **Root Directory** per service (`apps/api`, `apps/web`) and a custom build/start command that scopes the Turborepo run correctly.

For the API:
```
Build:  pnpm turbo build --filter=@myapp/api
Start:  node dist/index.js
```

For the web:
```
Build:  pnpm turbo build --filter=@myapp/web
Start:  (static output, served by Railway's static hosting)
```

The tricky part is that Turborepo's `build` task walks the dependency graph — it knows `@myapp/api` depends on `@myapp/shared-types` and `@myapp/parsing`, and it builds them in the right order. Railway just runs the command and gets a fully-built artifact without needing to know any of that internally.

## The Migration Problem

Thirty migrations don't run themselves. The question is *when* to run them.

The naive approach is a separate Railway service that runs `pnpm db:migrate` and exits. This works, but it's easy to get the ordering wrong — the migration service races with the API service at startup, and if the API wins, it crashes trying to query tables that don't exist yet.

The pattern that actually holds up: **run migrations inside the API's start command, before the server binds to a port**.

```bash
# Railway start command for the API service
node dist/scripts/run-migrations.js && node dist/index.js
```

Railway considers a deployment healthy once the process is running and the health check passes. Since `&&` means "only proceed if migrations succeeded," a failed migration exits the process, Railway marks the deployment as failed, and the old version keeps serving traffic. You get atomic deploys: migrations and code ship together, or neither does.

The one gotcha: your migration script needs to be idempotent. `CREATE TABLE IF NOT EXISTS` and conditional `ALTER TABLE` statements mean you can re-run migrations safely on a redeploy without blowing anything up. We'd been doing this since migration 001, so it was a non-issue — but it's the kind of thing that bites you if you didn't plan for it.

## Environment Variables

Railway's variable management is genuinely good. You set `DATABASE_URL` once on the PostgreSQL service and reference it in the API service with `${{Postgres.DATABASE_URL}}`. Railway interpolates it at deploy time. Same for the JWT secrets, CORS origin, and the new `AI_ENABLED` / `AI_API_KEY` variables we'd just added.

One thing worth knowing: Railway injects `PORT` automatically, and your server should read `process.env.PORT` rather than hardcoding 3001. We already did this, but it's a common trip-up.

## What We Learned

Monorepo deployments on Railway are straightforward once you accept that the platform wants *services*, not a single repo. Split your deploy units, scope your build commands with Turborepo's `--filter`, and fold migrations into the start command rather than treating them as a separate concern.

The bigger lesson: decisions you make early in a project — idempotent migrations, `process.env.PORT`, environment-based config — either pay dividends or become last-minute scrambles when deploy day finally arrives. Ours paid off.

