---
layout: post
title: "The Document That Broke Our Single-Pass Pipeline"
date: 2026-04-07 17:32:50 +0000
categories: [performance]
tags: [llm, document-parsing, pipeline-design, benchmarking]
excerpt: "We ran a head-to-head experiment between a single-call LLM parser and a multi-step pipeline on real vendor documents — and a 547-item price list exposed exactly where the single-pass approach falls apart."
---

We had a theory: the layout-analysis step in our document parsing pipeline was adding latency without adding value. So we built a single-pass alternative — one LLM call, extract everything at once — and ran both pipelines against six real vendor price sheets to find out.

The results were more interesting than a simple winner.

## The Two Pipelines

The standard pipeline has three steps. First, a layout agent looks at the document and maps its structure — which columns contain prices, which contain pack sizes, what format the SKUs are in. Then an extraction agent pulls the actual data using that map. Finally, a deterministic cleaner normalizes units, validates prices, and flags anything suspicious.

The single-pass pipeline skips the layout step entirely. One call, one prompt: "Here's the document, give me every product with its price, pack size, and SKU."

The appeal is obvious. Fewer round-trips, less latency, simpler code. And for small documents, it works exactly as well as the multi-pass approach — nearly identical item counts, nearly identical speed.

## Where It Broke

Then we hit a large format price list.

On smaller vendor sheets (60-80 items), both pipelines extracted the same number of items in roughly the same time. Sometimes single-pass was a few seconds faster. The layout step was genuinely adding nothing.

But one document — a full distributor catalog — contained around 550 line items across many pages. The standard pipeline got **547 items**. The single-pass pipeline got **115**.

That's a 79% miss rate. Not because the model misread anything. It just ran out of output budget before it finished the document. LLMs have a maximum output length, and a request that says "extract all products from this 40-page catalog" will hit that ceiling well before it's done.

The multi-step pipeline sidesteps this entirely because it processes the document page-by-page. Each extraction call covers a bounded chunk. The layout step isn't just overhead — on large documents, it's the thing that makes chunked processing possible.

## The Unexpected Reversal

There was one twist. On that same large document, pack size extraction was *better* in the single-pass pipeline: 63% of items had a pack size vs only 12% in the standard pipeline.

We haven't fully traced why. Our best guess is that the layout mapper is misidentifying the pack size column for that particular vendor's format — it's a wide catalog with an unusual column order — while the single-pass model, seeing the full row in context, infers pack size more reliably for the items it does process.

It's a reminder that "better coverage" and "better quality" can diverge. The multi-pass pipeline captured 4.7x more items overall, but it also introduced a column-mapping error that single-pass avoided.

## The Rule We Landed On

Scale is the differentiator, not format complexity.

For any document under roughly 100–130 items, single-pass is equivalent to multi-pass and marginally faster. For large catalogs, multi-pass is the only option that reliably captures everything — the single-pass ceiling is a hard constraint, not a tuning problem.

Our approach going forward: route small documents through single-pass, large documents through the multi-step pipeline. The routing heuristic is simple — estimate page count from the file before parsing, and pick the pipeline accordingly.

The layout step we thought was overhead turned out to be load-bearing. We just couldn't see it until we ran the experiment on a document large enough to expose it.
