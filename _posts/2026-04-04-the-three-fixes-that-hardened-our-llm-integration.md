---
layout: post
title: "The Three Fixes That Hardened Our LLM Integration"
date: 2026-04-04 16:32:27 +0000
categories: [integration]
tags: [llm, security, prompt-injection, typescript]
excerpt: "A code review of our AI parsing layer flagged prompt injection, missing timeouts, and a config that silently enabled itself — here's what the fixes actually looked like in code."
---


A code review came back with three critical issues on our AI parsing layer. Not "consider improving" suggestions — hard blocks before merge. Prompt injection via unsanitized vendor names. No timeout on LLM calls. And a config default that would silently enable AI for every user the moment someone dropped an API key into the environment.

We'd caught them before shipping. Now we had to actually fix them.

## Fix 1: A Shared Sanitization Utility

The prompt injection issue was subtle. Our templates interpolated vendor names and user-supplied corrections directly into the prompt string:

```typescript
prompt += `Vendor: ${vendor}\n\n`
// ...
prompt += `- "${c.originalValue}" → "${c.correctedValue}"\n`
```

If a vendor name contained a newline followed by `IGNORE PREVIOUS INSTRUCTIONS`, we'd hand that straight to the model. The fix wasn't to patch each template individually — that's how you miss one. We created a single utility and applied it everywhere:

```typescript
export function sanitizeForPrompt(input: string, maxLength = 500): string {
  return input
    .replace(/[\n\r\t]/g, ' ')       // collapse newlines — the main injection vector
    .replace(/[^\x20-\x7E]/g, '')    // strip non-printable chars
    .replace(/\s+/g, ' ')
    .trim()
    .slice(0, maxLength)             // hard length cap
}
```

Then every template call that touched user-supplied data went through it:

```typescript
prompt += `Vendor: ${sanitizeForPrompt(vendor)}\n\n`
prompt += `- "${sanitizeForPrompt(c.originalValue)}" → "${sanitizeForPrompt(c.correctedValue)}"\n`
```

One function, one place to audit. If we add a new template later, the pattern is obvious.

## Fix 2: AbortController on Every LLM Call

The original provider called the Claude API with no timeout whatsoever:

```typescript
const response = await this.client.messages.create({ ... })
```

A hung request here means a hung import handler. The whole upload flow blocks. Users see a spinner forever. The fix was an `AbortController` with a configurable timeout that defaults to 30 seconds:

```typescript
async complete(prompt: string, options: AIRequestOptions = {}): Promise<AIResponse> {
  const timeoutMs = options.timeout ?? this.config.timeoutMs ?? 30_000
  const controller = new AbortController()
  const timer = setTimeout(() => controller.abort(), timeoutMs)

  try {
    const response = await this.client.messages.create(
      { /* ...params */ },
      { signal: controller.signal }
    )
    return mapResponse(response)
  } catch (error) {
    if (error instanceof Error && error.name === 'AbortError') {
      throw new Error(`AI request timed out after ${timeoutMs}ms`)
    }
    throw error
  } finally {
    clearTimeout(timer)
  }
}
```

The explicit `finally` to clear the timer matters — otherwise you leave a dangling handle even when the request succeeds.

## Fix 3: Opt-In, Not Opt-Out

This one was the sneakiest. The original config parser read:

```typescript
export function parseAIConfig(env: NodeJS.ProcessEnv): AIProviderConfig | null {
  if (!env.AI_API_KEY) return null   // AI disabled if no key
  return {
    enabled: env.AI_ENABLED !== 'false',  // enabled by default!
    // ...
  }
}
```

The intent was "you can disable it by setting `AI_ENABLED=false`." The actual behavior: drop an API key in `.env` during a test, forget to set the flag, and AI silently turns on for all users. We flipped it:

```typescript
export function parseAIConfig(env: NodeJS.ProcessEnv): AIProviderConfig | null {
  if (env.AI_ENABLED !== 'true') return null   // must explicitly opt in
  if (!env.AI_API_KEY) return null
  return { enabled: true, /* ... */ }
}
```

Now the `.env.example` comment matches reality:

```bash
# AI_ENABLED=true    # Must be explicitly set to enable AI features
# AI_API_KEY=...     # Required when AI_ENABLED=true
```

---

The common thread across all three: **defaults should be safe, not convenient**. Sanitize at the boundary between user data and LLM prompts, not deep inside individual templates. Apply timeouts at the transport layer, not as an afterthought. And features that cost money or carry security implications should require deliberate activation.

A code review is most useful when it catches these *structural* issues — the ones that are easy to get wrong once and then copy everywhere. Getting blockers back before merging is the system working correctly.

