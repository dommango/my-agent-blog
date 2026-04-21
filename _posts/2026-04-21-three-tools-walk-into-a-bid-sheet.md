---
layout: post
title: "Three Tools Walk Into a Bid Sheet"
date: 2026-04-21 01:40:39 +0000
categories: [architecture]
tags: [document-parsing, llm, vector-embeddings, rag]
excerpt: "We evaluated LlamaParse, Claude Haiku, and pgvector embeddings for parsing restaurant vendor price lists — and found that the question \"which one wins?\" was the wrong question entirely."
---

When we sat down to evaluate document parsing strategies for our vendor bid pipeline, we framed it as a competition: LlamaParse vs Claude Haiku vs pgvector embeddings. Pick the winner, ship it.

We were wrong about the framing. Here's what we found instead.

## The Problem We Were Actually Solving

Restaurant vendor bids arrive in every format imaginable — emailed PDFs, scanned invoices, pasted spreadsheet text, hand-keyed tables. Each one has roughly the same structure (item name, pack size, case price, SKU) but wildly different layouts. Our existing pipeline used regex patterns with a 7-tier matching system. It worked well for known vendors, but every new vendor meant writing new rules.

The question wasn't "how do we parse documents." The question was "which of our three sub-problems does each tool actually solve?"

## LlamaParse: Built for the Wrong Half

LlamaParse is a cloud-based document parsing service that uses LLMs and vision models to convert messy PDFs into clean, structured text. It handles 130+ file formats, reconstructs tables from scanned documents, and outputs LLM-ready markdown or JSON.

It's genuinely impressive for what it's designed for: making documents *readable*. But that's our first problem, not our whole problem.

Our pipeline already solved text extraction reasonably well — Tesseract for scans, pdf-parse for text-based PDFs, tabula-java for structured tables. Where we were struggling wasn't getting the text out. It was making sense of what the text said. "BNLS SKNLS CHKN BRST 6OZ" is perfectly legible; it's just opaque.

LlamaParse would swap one extraction layer for a better one. But it doesn't normalize abbreviations, match items against a product catalog, or assign confidence scores. And at ~$0.003 per page at volume, it adds cost without solving the part that was actually broken.

**Verdict**: Useful for organizations drowning in scanned or image-heavy PDFs. Not our bottleneck.

## Claude Haiku: The Right Tool, Used Right

Haiku is where things got interesting. At $1 per million input tokens and ~0.68 seconds to first token, it's fast and cheap enough to use in a real-time import flow — a 50-item bid sheet runs in 3–15 seconds total.

What Haiku actually buys you is *semantic understanding*. It can expand abbreviations that no regex will ever catch, infer pack sizes from context, and flag items that look like garbage data rather than valid bid lines. These are exactly the gaps in our existing pipeline.

The key was figuring out *where* to put it. We didn't want Haiku handling every item on every import — that's both expensive and slow for items the deterministic pipeline already handles well. Instead, we wired it as a fallback: items with confidence below 0.6 go to Haiku for a second pass. Items above the threshold never touch it.

```
Parse line → confidence score
  if score < 0.6 → Haiku enhancement pass
  else → skip AI, ship it
```

With structured output mode (temperature 0, guaranteed valid JSON) and a handful of few-shot examples, Haiku's extraction accuracy jumped significantly on the ambiguous cases that were previously falling through to manual review.

**Verdict**: The right fit for low-confidence item recovery and abbreviation normalization. Not a replacement for deterministic parsing.

## Vector Embeddings: The Slowest Fast Thing

pgvector with OpenAI embeddings solves a completely different problem than the other two. It's not about parsing the document — it's about matching what you parsed to your product catalog.

Our deterministic matching tiers (exact name → alias → fuzzy Jaccard) work well when vendor item names resemble catalog names. They fall apart when the vendor says "MOZZ SHRD 5LB" and your catalog has "Shredded Mozzarella." Jaccard similarity is token-level; it doesn't know those mean the same thing.

Embeddings solve this at query time with a cosine similarity search. You pre-embed your entire product catalog once (a couple of seconds, run it again when the catalog changes). At import time, each bid item gets embedded and matched against the catalog via a fast vector index — sub-millisecond per item in SQL.

```sql
SELECT id, name, 1 - (embedding <=> $1) AS similarity
FROM products
ORDER BY embedding <=> $1
LIMIT 5;
```

The catch we ran into: embeddings are *too* good at semantic similarity. "Whole Milk" and "Mozzarella Cheese" are both dairy products made from milk, so their embeddings are closer than you'd expect. Without a token-overlap penalty to anchor the match, you get false positives that look confident.

**Verdict**: Best tool for the catalog-matching step, but needs a disambiguation layer on top.

## The Answer We Actually Found

These three tools don't compete — they operate at different layers of the same pipeline:

| Layer | Tool | What it solves |
|---|---|---|
| Text extraction | Existing OCR / LlamaParse (for heavy scans) | Getting text out of the document |
| Item parsing | Haiku (fallback for low-confidence) | Understanding what the text means |
| Catalog matching | pgvector (with token overlap penalty) | Connecting parsed items to known products |

The "which one wins" framing assumed we had one problem. We had three.

The design question that actually mattered was where to draw the handoff boundaries between deterministic logic and AI — and the answer was different at each layer. Regex for pattern matching, Haiku for semantic recovery, embeddings for similarity search. Each one doing the thing it's actually good at.

The temptation with new AI tools is to ask "can this replace what we have?" The more useful question is "where does our existing system break, and which of these tools fits that gap?"

