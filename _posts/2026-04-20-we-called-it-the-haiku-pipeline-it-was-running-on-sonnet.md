---
layout: post
title: "We Called It the Haiku Pipeline. It Was Running on Sonnet."
date: 2026-04-20 21:21:48 +0000
categories: [performance]
tags: [llm, cost-optimization, haiku, parsing]
excerpt: "We added token tracking to our quick parse pipeline and discovered every call was costing 3-4x more than expected — because the \"Haiku-optimized\" pipeline had been defaulting to Sonnet the whole time."
---


We spent weeks tuning our quick parse pipeline specifically for Haiku. Smaller prompts. Batched calls. Focused system instructions. The whole pipeline was designed around a mental model of "$0.001 per bid sheet."

Then we added token tracking — and watched a two-item parse ring up at **$0.013**.

That's not a Haiku number. That's a Sonnet number.

## How It Happened

The quick parse pipeline runs six AI steps on every text import: structure analysis, sanity checking, SKU recovery, pack size recovery, vendor detection, and name cleanup. Each step calls through a shared AI provider singleton that reads its model from an environment variable.

The environment variable was set to `claude-sonnet-4-6`.

Not because anyone made a deliberate decision. Because when we added the AI config block to `.env.example`, Sonnet was the working default for development, and nobody changed it when we later decided the quick parse steps were Haiku-appropriate work.

The singleton had been happily routing every lightweight call — "does this item look like garbage?", "what's the pack size on this line?" — through a model with 3x the input cost and 4x the output cost of Haiku. For months.

## The Token Math

Once we had visibility into what the pipeline was actually spending, the numbers were clarifying:

```
2-item quick parse (Sonnet @ $3/$15 per M):
  2,244 input tokens  → $0.0067
    432 output tokens → $0.0065
  Total: ~$0.013

Same parse on Haiku @ $0.80/$4 per M:
  2,244 input tokens  → $0.0018
    432 output tokens → $0.0017
  Total: ~$0.0035
```

3.75x more expensive per parse. At realistic import volume — dozens of bid sheets per week per restaurant — that compounds fast.

## Why Token Tracking Matters More Than You Think

The insight here isn't really about Haiku vs Sonnet. We already knew Haiku was cheaper. The insight is that **without per-call token tracking, you're flying blind on what your pipeline actually costs.**

We had cost estimates. We had architectural intent. What we didn't have was a feedback loop that would tell us when the intent diverged from reality. The model used in production is an environment variable, and environment variables drift.

Adding `onTokenUsage` callbacks to every pipeline step — accumulating input tokens, output tokens, and model name into the parse job record — took about two hours. The cost of *not* having it was months of overspending and no visibility into which steps were driving it.

## The Fix

Two things:

**1. Change the default.** Quick parse steps are deterministic text operations — sanity checking a list of food items, expanding abbreviations, recovering a SKU from pattern context. Haiku handles all of them with the same accuracy as Sonnet. The default for the quick parse singleton should be Haiku, with Sonnet reserved for agent-parse (which does PDF vision, where it actually earns its price).

**2. Make the model explicit at the call site, not just at config.** Relying on an env var to express "this pipeline should use a cheaper model" is fragile. The intent should live in the code:

```typescript
// Haiku for quick deterministic ops — explicit, not configurable
const result = await provider.complete(prompt, {
  model: 'claude-haiku-4-5-20251001',
  temperature: 0,
})
```

If someone sets `AI_MODEL=claude-opus-4` in their environment to test something, the quick parse pipeline shouldn't silently switch to the most expensive model available. The model choice for a given step is part of that step's design.

## The Lesson

Architecture docs say "use Haiku for lightweight agents." Code comments say "optimized for Haiku." But the actual model is wherever the environment variable points.

If you're building a multi-step AI pipeline with cost sensitivity, token tracking isn't a nice-to-have — it's how you verify that the system you built is the system that's running.

