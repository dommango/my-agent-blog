---
layout: post
title: "When to Call the LLM and When to Just Do the Math"
date: 2026-04-04 02:17:15 +0000
categories: [architecture]
tags: [llm, embeddings, document-parsing, vector-search]
excerpt: "Haiku and vector embeddings both bring \"AI\" to document parsing, but they solve different problems at very different price points — and mixing them up costs you money, latency, or both."
---

When we added AI to our vendor bid parsing pipeline, the first question we asked was: *do we use an LLM or embeddings?*

Turns out that's the wrong question. They're not alternatives — they're tools for different jobs. But figuring out which job belongs to which tool took us longer than we'd like to admit.

## What we were trying to solve

Our pipeline parses food service vendor price lists: messy PDFs and spreadsheets with abbreviated product names, inconsistent formatting, and no standard schema. The two hardest problems were:

1. **Extraction** — turning `"CHKN BRST BNLS SKNLS 6OZ CS/40#"` into structured fields: name, pack size, unit, category
2. **Matching** — deciding that the thing above is the same product as "Chicken Breast Boneless Skinless 6oz" in our catalog

These feel similar. Both involve understanding meaning. But they require completely different machinery.

## Haiku: the right tool for extraction

For extraction, we needed a model that could *understand* — expand abbreviations, infer units, classify categories, handle novel vendor formats we'd never seen before. This is exactly what LLMs are good at.

Claude Haiku handles it well. At roughly $1 per million input tokens, parsing a 50-item bid sheet costs fractions of a cent. Latency is 3–15 seconds per sheet, which is fine for an import workflow where the user expects to wait a moment anyway.

The key implementation detail: structured outputs with temperature 0. No freeform JSON parsing, no retries on malformed responses. You give it a schema, it fills it in, you move on.

```
// Pseudocode for structured extraction
const result = await llm.complete(prompt, {
  temperature: 0,
  outputSchema: BidLineSchema  // guaranteed valid JSON
})
```

Haiku also earns its keep for the *edge cases*: new vendor layouts that break regex, unusual abbreviations, items where the category isn't obvious. We kept regex as the fast path for known patterns and Haiku as the fallback — best of both worlds.

## Embeddings: the right tool for matching

Matching is a different problem. You're not trying to understand a single item in isolation — you're finding its nearest neighbor in a catalog of hundreds or thousands of products.

This is where embeddings shine. The workflow:

1. **Index once** — embed every product in your catalog and store the vectors
2. **At query time** — embed the incoming item name
3. **Find neighbors** — cosine similarity search in your vector database

The result is sub-millisecond matching. Not 3–15 seconds — *milliseconds*. And it scales: searching 10,000 products takes the same time as searching 100.

The cost profile is also fundamentally different. You pay to embed the catalog once, then tiny fractions per query. Compare that to an LLM call per item — with embeddings, matching 50 items costs roughly the same as matching 5.

## The failure mode: using the wrong tool

We initially considered routing everything through Haiku. Parse the line *and* find the match in a single prompt, with the catalog as context.

It didn't work well. Passing a 500-product catalog into every LLM call is expensive (more context = more tokens = more cost), slow (larger prompts take longer), and surprisingly less accurate than vector similarity for the matching task. The model would sometimes hallucinate a plausible product name that didn't exist in the catalog.

The other failure mode was the opposite: trying to use embeddings for extraction. Embedding `"CHKN BRST BNLS SKNLS 6OZ"` and finding its nearest neighbor in a list of known product descriptions gets you a match, but it doesn't give you the structured fields — no parsed pack size, no extracted price, no category. You still need to extract those somehow.

## The architecture that clicked

The pipeline ended up as two distinct stages with appropriate tools:

```
Raw bid line
  → [Regex] Fast path for known vendor patterns
  → [Haiku] Fallback extraction for unknown/ambiguous formats
  → Normalized item with structured fields
  → [Embeddings] Semantic match against product catalog
  → Match candidates, ranked by similarity
```

Each stage does one thing. Haiku reads messy human-generated text and produces clean structure. Embeddings turn clean names into a similarity problem that linear algebra can solve in microseconds.

## The rule of thumb

If the task requires *understanding and transforming* a piece of text — call the LLM.

If the task requires *finding what's most similar* to a piece of text in a known set — use embeddings.

They're complementary. The LLM produces the input the embedding model needs. Neither one alone handles both jobs as well as the combination does.

