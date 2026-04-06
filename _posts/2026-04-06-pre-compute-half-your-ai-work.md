---
layout: post
title: "Pre-Compute Half Your AI Work"
date: 2026-04-06 17:05:54 +0000
categories: [architecture]
tags: [embeddings, llm, parsing, vector-search]
excerpt: "In a production parsing pipeline, the key architectural question isn't LLMs vs embeddings — it's figuring out which AI work belongs offline and which belongs at import time."
---

When we were designing the AI layer for our bid parsing pipeline, we spent a lot of time asking the wrong question: "LLMs or embeddings?"

The better question turned out to be: **when does this computation need to happen?**

## The Pipeline Problem

We're parsing vendor bid sheets — scanned PDFs, pasted text, spreadsheets — and trying to match each item ("BNLS SKNLS CHKN BRST 6OZ") to a product in our catalog. The catalog has a few hundred products. Bid sheets have 30–100 items each. Imports happen throughout the day.

The naive AI approach is to call an LLM for every match decision. That works at demo scale. At production scale, it's slow and expensive — not catastrophically, but annoyingly. Even with Haiku at $1/million tokens, you're paying per import, per item, per request.

The insight that changed our architecture: **our catalog doesn't change very often**. Products get added or edited occasionally, but the catalog is mostly stable between imports. So why would we re-examine it on every single import?

## Two Phases, Two Tools

We split the AI work into two phases based on when the data changes.

**Phase 1: Catalog build (offline)**

When a product is added or updated, we generate a vector embedding — a 1,536-dimension numerical representation of its meaning — and store it in the database. This happens once per product change, not once per import.

```sql
-- Stored on write, queried at import time
ALTER TABLE products ADD COLUMN embedding vector(1536);
CREATE INDEX ON products USING hnsw (embedding vector_cosine_ops);
```

The result is that our entire product catalog lives pre-computed in the database, indexed for fast similarity search. No LLM calls needed at query time.

**Phase 2: Import (online)**

When a bid sheet arrives, we embed each line item in a single batch call — 50 items in, 50 vectors out. Then we run a SQL query to find the nearest product vector for each. The whole matching step completes in milliseconds.

The LLM shows up only for the hard cases: items where no match scored above our confidence threshold, or where the top-three candidates are close enough to be ambiguous. At that point, we send those specific items to Haiku for judgment — not the whole batch.

## The Token Overlap Problem

Pure vector similarity has a production failure mode that doesn't show up in benchmarks.

Mozzarella and whole milk are semantically related (both dairy, both from cows, both white). Their embeddings are close. So "MOZZ SHRD 5LB" can match "Whole Milk" if your catalog happens to have a gap near "Mozzarella Shredded."

We hit this in testing. The fix was a **token overlap penalty**: before accepting a vector match, we compute Jaccard similarity on the tokenized product names. If the vector similarity is high but the token overlap is low, we down-score the match. This prevents cross-category contamination without throwing away the semantic signal.

```
final_score = vector_similarity * (1 - α * (1 - token_overlap))
```

The parameter α controls how much to trust tokens vs semantics. We tuned it on a holdout set. The key insight is that neither signal alone is reliable — tokens fail on abbreviations ("BNLS" vs "Boneless"), vectors fail on related-but-different items. Together they're much more robust.

## The Fallback Chain

In production, the matching pipeline runs in order of cost:

1. **Deterministic** — SKU lookup, exact name match, known aliases. Free.
2. **Vector similarity** — pgvector cosine search with token overlap penalty. Fast, cheap.
3. **LLM normalization** — Haiku expands abbreviations, re-runs deterministic matching. Slower, but targeted.
4. **LLM judgment** — Haiku picks from top-3 candidates when vector scores are close. Last resort.

Most items never leave step 1 or 2. The LLM path handles maybe 15% of items in practice — the ones that actually need it.

## What This Changes

The temporal split matters beyond just performance. It changes how you think about cost, reliability, and deployment.

**Cost becomes predictable.** Catalog embedding costs scale with catalog size, not import volume. A hundred imports a day costs the same in AI as one import a day, as long as your catalog isn't changing constantly.

**Failures degrade gracefully.** If the LLM is unavailable, deterministic and vector matching still work. The system produces lower-confidence results rather than failing entirely.

**Caching becomes effective.** LLM calls are cached by a hash of the normalized input. The same vendor abbreviation ("BNLS SKNLS CHKN BRST") produces the same cache hit regardless of which bid sheet it came from. Because we're only calling the LLM on the genuinely hard cases, the cache hit rate is high.

The broader lesson: before adding AI to a pipeline step, ask whether the inputs to that step change on every request or just occasionally. If occasionally — embed them once and query fast. The LLM is still there for the cases that need it. It just isn't doing work it doesn't have to.

