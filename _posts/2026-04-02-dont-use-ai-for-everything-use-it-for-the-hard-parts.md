---
layout: post
title: "Don't Use AI for Everything. Use It for the Hard Parts."
date: 2026-04-02 16:45:10 +0000
categories: [architecture]
tags: [document-parsing, llm, production, saas]
excerpt: "When we evaluated AI tools for our document parsing pipeline, the real question wasn't which tool to pick — it was figuring out exactly which steps actually needed AI at all."
---


We had a working bid sheet parser. It handled maybe 70% of the vendor formats we'd seen. For the other 30%, it produced garbage — wrong prices, swapped fields, missing SKUs. The obvious fix felt like: add AI.

But "add AI" isn't a plan. We needed to figure out *where* in the pipeline AI actually belonged.

## The Evaluation Trap

When we started researching options — third-party document parsing services, vision models, embedding APIs — we nearly fell into a trap that a lot of teams hit. We were evaluating tools before we'd mapped the problem.

The question we started with was: "Should we use a specialized document parsing service or call an LLM directly?" That's the wrong question. The right question is: "What's actually failing, and why?"

When we mapped our pipeline step by step, the failures clustered in predictable places:

1. **Vendor detection** — regex knew six vendors, failed on all others
2. **Format structure** — unknown column layouts broke our field extractor
3. **Abbreviations** — `BNLS SKNLS CHKN BRST 6OZ` doesn't fuzzy-match to `Chicken Breast Boneless Skinless`
4. **Scanned PDFs** — Tesseract struggles on tabular layouts
5. **Low-quality SKUs** — some vendors just don't have them

Once we had that list, the tool choice got much easier.

## The Layered Approach

We ended up with a nine-step pipeline. The key design principle: **deterministic methods run first, AI only handles what they can't.**

```
1. Vendor detection       → pattern match → LLM fallback
2. Structure analysis     → LLM reads layout, returns profile
3. Profile-guided parse   → use profile for field extraction
4. Line parsing           → regex (fast, cheap, reliable)
5. Sanity check           → LLM flags garbage items
6. SKU recovery           → LLM fills missing SKUs
7. Pack size recovery     → LLM fills missing pack sizes
8. Name cleanup           → LLM expands abbreviations
9. Field validation       → reject items missing price
```

Steps 4 and 9 — the highest-volume work — never touch the LLM. They're pure logic. Steps 2, 5, 6, 7, and 8 use the LLM surgically, only when the data doesn't yield to pattern matching.

The result: for a typical 75-item bid sheet, maybe 15-20 items go through any AI step at all.

## Why We Picked a Small Model Over a Specialized Service

The specialized document parsing services are impressive. They handle 130+ formats, support tables and charts, output clean structured data. For a general-purpose document pipeline, they'd be a strong choice.

But our problem wasn't general. We needed:
- A specific output schema (our product types, not generic JSON)
- Contextual judgment (is this a reasonable price for chicken? does this SKU match the vendor's pattern?)
- Tight integration with our existing confidence scoring
- Per-step control so we could measure where AI helped and where it didn't

A small, fast model called with structured output instructions gave us all of that. Cost turned out to be essentially irrelevant at our scale — a few dollars a month. Latency mattered more: keeping AI calls under 5 seconds per import required batching and caching.

## The Caching Insight

One thing we didn't anticipate: the same vendor sends nearly identical bid sheets month to month. Prices change, quantities shift, but the product names are the same.

Once we added a cache keyed on the normalized input text, the effective AI call rate dropped dramatically. Name normalization for `BNLS SKNLS CHKN BRST` only needs to happen once. After that, it's a cache hit.

This also meant the "cost at scale" math changed. The expensive case isn't the ongoing steady state — it's the first import from a new vendor. After that, it's almost free.

## What We Learned

A few things worth carrying forward:

**Map failures before evaluating tools.** The tool choice follows naturally from understanding what's actually broken.

**Prefer a fast deterministic path.** AI in a parsing pipeline should feel like a specialist you consult on hard cases, not an assembly line worker handling every item.

**Structured outputs are non-negotiable in production.** Freeform LLM responses that need to be parsed are a reliability tax. Force the model to return valid JSON from the start.

**Measure where AI actually helps.** We logged confidence scores before and after AI enhancement. Some steps moved the needle significantly. Others barely changed the output — and we turned those off.

The goal was never to build an AI-powered parser. It was to build a parser that worked reliably on vendor formats we'd never seen before. AI was one tool toward that goal — and knowing exactly where to apply it made all the difference.

