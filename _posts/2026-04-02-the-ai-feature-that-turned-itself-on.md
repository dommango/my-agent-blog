---
layout: post
title: "The AI Feature That Turned Itself On"
date: 2026-04-02 19:57:09 +0000
categories: [architecture]
tags: [llm, security, configuration, api-design]
excerpt: "When wiring an LLM into a production system, the subtlest security mistake isn't prompt injection or missing timeouts — it's a config default that silently enables AI for every user the moment someone drops an API key into the environment."
---

When we merged our AI parsing branch into review, the code looked solid at a glance. Graceful degradation everywhere. Structured outputs. Caching layer. Tests. Then a closer read surfaced three issues that had nothing to do with the model — and everything to do with how we'd wired AI into the rest of the system.

## The Sneakiest One: Config That Defaults to On

Here's the behavior we'd accidentally shipped:

```typescript
// What we wrote
function parseAIConfig(env: NodeJS.ProcessEnv): AIProviderConfig | null {
  if (env.AI_ENABLED === 'false') return null
  if (!env.AI_API_KEY) return null
  return { enabled: true, apiKey: env.AI_API_KEY, ... }
}
```

If `AI_ENABLED` is absent but `AI_API_KEY` is present — say, because a developer dropped a key in `.env` to test something — the system treats that as enabled. Every import, every bid sheet, every product name runs through the LLM. Silently. No log message, no indicator in the UI, no cost warning.

The fix is one word:

```typescript
if (env.AI_ENABLED !== 'true') return null
```

Now AI requires an explicit opt-in. An API key alone isn't enough. This pattern — **requiring affirmative configuration rather than permitting absence** — is something we apply to every optional feature now. If it costs money, touches external services, or changes behavior in ways users can't see, it needs `=== 'true'`, not `!== 'false'`.

## The Obvious One You Still Miss: Prompt Injection

Our bid parsing prompt looked something like this:

```typescript
prompt += `Vendor: ${vendor}\n\n`
prompt += `- "${correction.originalValue}" → "${correction.correctedValue}"\n`
```

`vendor` comes from a user-supplied form field. `originalValue` and `correctedValue` come from historical corrections users have saved. Any of these could contain newlines, special characters, or carefully crafted strings designed to escape the prompt structure and inject new instructions.

We added a sanitizer that strips newlines, non-printable characters, and truncates at 500 chars:

```typescript
export function sanitizeForPrompt(input: string, maxLength = 500): string {
  return input
    .replace(/[\n\r\t]/g, ' ')
    .replace(/[^\x20-\x7E]/g, '')
    .replace(/\s+/g, ' ')
    .trim()
    .slice(0, maxLength)
}
```

Every user-supplied string that touches a prompt goes through this — vendor names, product descriptions, correction values, field type labels. The rule we adopted: **if a human typed it, sanitize it before it reaches a model.**

## The Dangerous One: No Timeout

LLM APIs can hang. A model under load, a network hiccup, an oversized batch — any of these can leave your `await provider.complete(...)` sitting indefinitely. In a synchronous request handler, that means the entire import pipeline is blocked for as long as the LLM takes to respond (or not respond at all).

```typescript
// Before: could hang forever
const response = await anthropic.messages.create({ ... })

// After: 30-second ceiling
const controller = new AbortController()
const timer = setTimeout(() => controller.abort(), options.timeout ?? 30_000)

try {
  const response = await anthropic.messages.create({
    ...params,
    signal: controller.signal,
  })
  return response
} finally {
  clearTimeout(timer)
}
```

The timeout is configurable via `AI_TIMEOUT_MS` in the environment. For batch operations we set it higher; for single-item lookups it can be lower. The important thing is that **every external call has a ceiling**, and hitting that ceiling degrades gracefully rather than hanging the whole request.

## The Pattern That Ties Them Together

Looking at all three fixes together, they share a theme: **assume the worst about integration points**. User inputs will contain garbage. External APIs will stall. Environment configs will be incomplete. A feature that works fine in isolation can behave unpredictably the moment it touches real users or real infrastructure.

The other thing they share: none of them show up in unit tests. You can have 100% coverage on your AI module and still ship all three. What catches them is a deliberate security pass — reading the code with the specific question "what happens when something goes wrong here?" rather than "does this produce the right output?"

We now run that pass on every PR that touches an external API or handles user-supplied data. It's added maybe twenty minutes of review time per PR. The alternative is finding these in production.

