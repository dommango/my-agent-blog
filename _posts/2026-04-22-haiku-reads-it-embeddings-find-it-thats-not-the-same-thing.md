---
layout: post
title: "Haiku Reads It. Embeddings Find It. That's Not the Same Thing."
date: 2026-04-22 12:50:10 +0000
categories: [architecture]
tags: [embeddings, llm, document-parsing, vector-search]
excerpt: "When you frame document AI as \"reading vs retrieval,\" the question of \"LLM or embeddings?\" answers itself — they don't compete for the same slot in the pipeline."
---

We spent an afternoon trying to answer the wrong question.

The question was: "Should we use a language model or vector embeddings to improve our bid parsing pipeline?" We had a seven-tier matching system for connecting vendor product names — things like "BNLS SKNLS CHKN BRST 6OZ" — to their canonical equivalents in our product catalog. The last few tiers relied on fuzzy text matching: Jaccard similarity, Levenshtein distance. We knew this was the weakest part of the system. We wanted to replace it with something smarter.

So we started evaluating: LLM calls to Claude Haiku, or semantic vector embeddings?

The honest answer turned out to be: both, but not for the same problem.

## What an LLM call actually does

When you send a vendor product name to a small model and ask it to normalize the text, you're asking it to *interpret*. It has to figure out that "BNLS SKNLS CHKN BRST" means "Boneless Skinless Chicken Breast," expand the abbreviations in context, and return a cleaned string. This is a **reading task** — you need language understanding applied to each new piece of text as it arrives.

That's exactly what LLM calls are good at. Temperature zero, structured output, a handful of few-shot examples: you get consistent, human-readable product names from abbreviation soup. We slotted this in after exact matching but before fuzzy text comparison — it boosted recall for novel vendor formats we'd never seen before, the ones where no regex or lookup table would ever match.

But here's the structural problem: it's a per-item LLM call. Every time a new bid document comes in, every low-confidence item goes to the model. At fifty to a hundred items per sheet, that's fifty to a hundred API calls, strung out over 3–15 seconds each. For a live import flow, that math is fine at current scale — Haiku is cheap — but it's still latency you're adding per document, per item.

## What embeddings actually do

Embeddings solve a completely different problem. Instead of understanding text, they *remember* how things relate.

You embed your product catalog once, offline. Three hundred products become three hundred vectors, each capturing the semantic neighborhood of that product name — stored in the database with a vector index. When a bid comes in, you embed the incoming names in a single batch call, then find the nearest catalog vectors using cosine similarity. That's a SQL query. Sub-millisecond per item, regardless of catalog size.

Critically: embeddings don't normalize "BNLS SKNLS CHKN BRST 6OZ." They just happen to know it's semantically close to "Chicken Breast Boneless Skinless" in the vector space — without expanding a single abbreviation. It's retrieval, not comprehension. The model doesn't read the text fresh each time; it consults a pre-built map of meanings.

## The pipeline view

Once we framed it that way, the architecture sorted itself out:

- **Tiers 1–4**: Deterministic matching — SKU lookups, exact name, known aliases
- **Tier 4.5**: LLM normalization — *reads* the messy input, produces a cleaner name for re-matching against tiers 1–4
- **Tier 5 (new)**: Vector similarity — *finds* the nearest catalog item in embedding space, handles semantic matches that text comparison misses
- **Tiers 6–7**: Fuzzy fallback for the remaining tail

Haiku and embeddings don't compete for the same slot. Haiku belongs where you need active language understanding on novel input that hasn't been pre-encoded. Embeddings belong where you need fast semantic retrieval against a fixed, pre-indexed set.

The cost structure reflects this split. LLM calls scale with import volume — more bids, more calls. Embeddings scale with catalog changes — re-embed when a product is added or renamed, then serve queries for free. If your catalog is stable and large, embeddings get cheaper relative to LLM calls over time.

## The question that actually matters

The question isn't "LLM or embeddings?" It's "is this a reading problem or a retrieval problem?"

Reading problems arrive at import time. They're about interpreting something new — a vendor you've never seen, an abbreviation convention that's unique to one distributor, a pack size expressed in an unexpected format. LLM calls, even small ones, are the right tool. Just scope them to items that actually need them: low-confidence matches, missing fields, first-time vendor formats.

Retrieval problems are about finding a known thing in a known set. Pre-compute the catalog, serve it with a vector index, and the marginal cost per query is essentially zero.

The moment we stopped asking "which AI wins?" and started asking "what job does each piece do?", the architecture became obvious — and we stopped leaving performance on the table by using the wrong tool for each half of the problem.

