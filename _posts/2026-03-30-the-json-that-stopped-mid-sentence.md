---
layout: post
title: "The JSON That Stopped Mid-Sentence"
date: 2026-03-30 23:54:25 +0000
categories: [debugging]
tags: [json, llm, parsing, debugging]
excerpt: "Our AI parsing pipeline returned 0 items on every request — not because the model failed, but because it ran out of room to finish the JSON it was writing."
---

We wired up an AI-powered parsing pipeline, ran it against a real vendor price list, and got this back:

```
Items: 0
Summary: { "totalLines": 0, "validLines": 0, ... }
```

Zero items. Every time. The request completed, the model responded — but nothing came through. Here's what was actually happening, and the four fixes that got us to 79 items extracted cleanly.

## The Investigation

The first clue was in the server logs. We added a line to capture the raw AI response before parsing:

```
[data-extractor] Page 1 raw response length: 20397
[data-extractor] Page 1: JSON parse failed
```

Twenty thousand characters of response — and a parse failure. That's not an empty response. Something was in there, it just wasn't valid JSON.

We captured the start and end of the content:

```
STARTS_WITH: "```json\n{\n  \"rows\": [\n    {\n      \"sku\": \"5BEA\", ..."
ENDS_WITH:   "... \"rawText\": \"SPR1A LOOSE BRUSSEL SPROUT 22 LB CASE * 82.25\""
```

The response started clean. It ended mid-object — no closing `]`, no closing `}`, no closing code fence. The model had simply run out of space and stopped writing.

## Fix 1: The Token Budget Was Too Small

The data extractor was configured with `maxTokens: 8192`. For a single-page vendor PDF with 79 line items, each rendered as a JSON object with 8–10 fields, that budget ran out around item 60. The model didn't error — it just stopped.

We doubled it:

```typescript
const options: AIRequestOptions = {
  maxTokens: 16384,  // was 8192
  temperature: 0,
  ...
}
```

That alone solved the zero-items problem. But we wanted the pipeline to be resilient even when truncation happens on unusually long documents.

## Fix 2: Truncated JSON Repair

When the model runs out of tokens mid-response, you get valid JSON up to a point — then it just stops. The structure is intact up to the last complete item; after that, it's garbage.

Our `parseJsonResponse` helper already stripped markdown code fences. We added a fallback that finds the last complete array element and closes the brackets:

```typescript
export function parseJsonResponse<T>(content: string): T | null {
  const cleaned = stripFences(content)

  // Try clean parse first
  try {
    return JSON.parse(cleaned) as T
  } catch {
    // May be truncated — salvage completed items
    return tryRepairTruncatedJson<T>(cleaned)
  }
}

function tryRepairTruncatedJson<T>(content: string): T | null {
  // Find the last complete object boundary (closing brace + optional comma)
  const lastComplete = content.lastIndexOf('},')
  if (lastComplete === -1) return null

  const salvaged = content.slice(0, lastComplete + 1) + ']}'
  try {
    return JSON.parse(salvaged) as T
  } catch {
    return null
  }
}
```

Now if a response is truncated, we recover whatever completed items the model did write instead of discarding everything.

## Fix 3: The Wrong Model ID

Separately, we noticed the production environment wasn't picking up the configured model. The code defaulted to a model ID that didn't quite match what the API expected — a version suffix difference that's easy to get wrong when model names change.

We stopped relying on the default and set the model ID explicitly in the environment:

```
AI_MODEL=claude-sonnet-4-20250514
AI_VISION_MODEL=claude-sonnet-4-20250514
```

A small thing, but if the fallback model is lower-capability than you expect, extraction quality quietly degrades without any obvious error.

## Fix 4: Price Extraction Fallback

Even after the other fixes, some items came through with `casePrice: null`. The model extracted everything else correctly but missed the price — usually when it appeared at the end of a line in an unusual format like `82.2400` with extra decimal places.

We added a trailing-number fallback in the attribute cleaner:

```typescript
if (row.casePrice === null) {
  const trailingPrice = row.rawText.match(/\s(\d+\.\d{2,4})\s*$/)
  if (trailingPrice) {
    casePrice = parseFloat(trailingPrice[1]!)
    priceInterpretation = 'extracted_from_raw'
  }
}
```

When the AI doesn't return a price, we try to extract a trailing decimal number from the raw text the model was given. It's not as robust as the model doing it, but it catches the common case where a price just happens to be at the end of the line.

## The Result

After all four fixes: 79 items extracted, prices populated, one page processed cleanly.

The lesson isn't really about JSON or token budgets specifically. It's that **a silent zero is the worst kind of failure** — nothing errored, nothing crashed, the pipeline just quietly produced nothing. Logging the raw response length before trying to parse it was what broke the case open. If you're calling an LLM and getting empty results, check the response content first. The model probably said something.

