---
layout: post
title: "Haiku vs Sonnet for Document Parsing: What the Data Actually Says"
date: 2026-04-14 18:21:23 +0000
categories: [performance]
tags: [claude, document-parsing, llm, benchmarking]
excerpt: "We ran both models head-to-head on six real vendor bid sheets and found that the choice of model matters far less than you'd expect — with one instructive exception."
---

When you're building an LLM-powered parsing pipeline, "which model should I use?" feels like a foundational architectural decision. We assumed Sonnet would be meaningfully better than Haiku for parsing structured vendor price sheets — richer names, fewer errors, better abbreviation expansion. So we ran a controlled experiment across six real documents and compared them directly.

The results were more nuanced than we expected.

## The Setup

We ran two pipelines in parallel on the same set of documents — vendor bid sheets from produce distributors, meat suppliers, and dairy farms. One pipeline used Haiku with a standard single-pass prompt. The other used Sonnet with a page-chunked approach (one LLM call per PDF page, results merged and deduplicated). We captured every extracted item as JSON and compared them by SKU, then by normalized name when SKUs were absent.

Six documents. Hundreds of items per file. We looked at three things: item recall (how many items each approach captured), name normalization quality (which model expanded abbreviations more accurately), and pack size specificity.

## Item Count: A Tie Everywhere Except One

On five of the six documents, both models extracted exactly the same number of items. Same SKUs, same prices, zero disagreement on what constituted a line item.

The exception was a large meat supplier price list — 547 items across many pages. Haiku got 515. Sonnet got 545. That's **30 extra items**, and they were real: lamb racks, specialty sausages, hot dogs that had simply disappeared from the Haiku output.

Here's the thing: those 30 missing items weren't a Haiku intelligence problem. They were a **token limit problem**. When you feed a 547-item document to an LLM in a single call, you hit output token ceilings and the model just... stops. Earlier in our work we'd already discovered that a page-chunked pipeline recovers more items on large documents than any single-pass approach, regardless of model. This experiment confirmed it again. The "Sonnet advantage" on that document was really a pipeline architecture advantage.

## Name Quality: Sonnet Is Fuller, Haiku Is Sometimes Cleaner

This is where the models actually diverged in meaningful ways.

Haiku would occasionally truncate or mangle abbreviations it couldn't confidently resolve. For a Swedish dairy farm's butter line, it produced "Grams Salt Butter" where Sonnet correctly gave "Grass-Fed Salt Butter." It clipped "Niman Ranch" to "Niman Ran" and "Clabber Girl" to "Clabber G." Small errors, but the kind that break fuzzy matching downstream.

Sonnet tended toward completeness — "Brie Wheel Kilo," "Reggiano Parmesan 1/4 Whole," "Gouda Cheese Whole Wheel" — pulling in pack-size context that the field extraction step would have to separate back out. Sometimes that was useful. Sometimes it was noise.

There was one genuine Haiku error we caught: it labeled a pastry item as a "Lamington" when the actual product was a croissant. Sonnet got it right. One Sonnet error in the other direction: it confused two separate items on a single SKU cluster, assigning a French fry description to what was actually a red pepper product. Haiku got that one right.

Neither model was systematically more accurate. They failed on different items.

## Prices: Completely Identical

This was the clearest finding. Across 959 matched item pairs, there was not a single price disagreement between Haiku and Sonnet. Not one. Price extraction from vendor documents — stripping currency symbols, parsing decimal formatting, handling "per lb" vs "per case" context — appears to be well within Haiku's capability ceiling. You are not paying for Sonnet to get better prices. You're getting the same prices at 3x the cost.

## What This Means in Practice

If your documents are small (under ~200 items), both models produce equivalent results. Use Haiku. The cost difference compounds quickly at scale — we were looking at roughly $0.03 per document with Haiku vs $0.09 with Sonnet at our volumes, which doesn't sound like much until you're processing thousands of imports per month.

If your documents are large, the answer isn't "use Sonnet." The answer is **chunk your documents**. Split PDFs by page, run the same Haiku prompt on each chunk in parallel, then deduplicate on SKU. This approach is faster (parallelism beats single-call latency), cheaper (Haiku), and captures more items (no output token ceiling).

The one place Sonnet earns its keep is name normalization for heavy abbreviation loads — vendor sheets where every item is a cryptic shorthand. Even then, the gap is smaller than you'd expect, and you can close much of it with good few-shot examples in your Haiku prompt.

The model selection question turned out to be less interesting than the architecture question. Chunking wins. Every time.

