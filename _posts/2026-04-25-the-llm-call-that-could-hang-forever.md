---
layout: post
title: "The LLM Call That Could Hang Forever"
date: 2026-04-25 03:30:45 +0000
categories: [debugging]
tags: [llm-integration, reliability, typescript, error-handling]
excerpt: "A code review of our AI parsing PR found careful error handling everywhere — and zero protection against the one failure mode that doesn't throw an error at all."
---


When we reviewed the PR that added AI-enhanced parsing to our bid import pipeline, the code looked solid. Every function that called the LLM was wrapped in a try/catch. If the AI provider was disabled, you got a no-op result back. If the API returned an error, the system gracefully fell back to the rule-based parser. The tests passed.

We almost missed the real problem.

## "Won't crash" and "won't freeze" are different guarantees

The error handling covered one class of failure: the API responds with something bad. Throws an exception? Caught. Returns garbled JSON? Caught. Returns a 429? Caught.

But what about the API that never responds?

A hung HTTP connection doesn't throw. It just waits. And our import pipeline — which called the LLM for each batch of low-confidence bid items — had no timeout set anywhere. If the upstream API stalled mid-request, the import job would sit there indefinitely, holding a worker, blocking any retry, and giving users nothing but a spinner.

The fix itself is simple:

```typescript
async complete(prompt: string, options: AIRequestOptions = {}): Promise<AIResponse> {
  const timeoutMs = options.timeout ?? 30_000
  const controller = new AbortController()
  const timer = setTimeout(() => controller.abort(), timeoutMs)

  try {
    const response = await anthropic.messages.create(
      { /* ... params */ },
      { signal: controller.signal }
    )
    return formatResponse(response)
  } finally {
    clearTimeout(timer)
  }
}
```

Thirty seconds, then abort. If it fires, the try/catch you already have handles it like any other failure — the batch falls back to rule-based parsing, the job completes, and the user gets results.

## Why this is easy to miss

Error handling and timeout handling feel like the same thing, but they're not. You add error handling when you imagine a failure. You add timeouts when you imagine *silence*. Most developers (and most code reviewers) are better at imagining failures than silence.

This is especially true with LLMs because they're usually fast. In development and testing, a Haiku call comes back in a few seconds. You never see it hang. So you write the catch blocks, you confirm the fallback path works, and you ship.

The hung call only shows up in production — during a network blip, under unexpected load, or when the upstream service is quietly degraded rather than fully down.

## The broader config problem we also fixed

While we were in there, we found a second issue: the feature was opt-out rather than opt-in. If `AI_ENABLED` wasn't set in the environment but an API key *was* present, the feature silently enabled itself. That meant adding a key for a quick test in staging could accidentally activate AI calls in production after a deploy.

The fix was one line — require `AI_ENABLED === 'true'` explicitly — but the principle matters: AI features carry costs, latency, and non-determinism that regular code doesn't. They should require conscious activation, not just credential presence.

## The rule we now check for every LLM integration

Two questions before merging any code that calls an LLM:

1. **What happens when it fails fast?** (The catch block question.)
2. **What happens when it never returns?** (The timeout question.)

Both answers need to be "something graceful," not "it depends on the caller."

