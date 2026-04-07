---
layout: post
title: "The Dumber Pipeline Won"
date: 2026-04-07 18:49:24 +0000
categories: [performance]
tags: [pdf-parsing, llm, benchmarking, document-ai]
excerpt: "When we benchmarked three PDF parsing strategies against real vendor price lists, the approach with no layout analysis and no whole-document context extracted the most items — by just splitting the PDF into pages and running the same prompt on each one."
---

We had three ways to turn a vendor price list into structured data.

**Pipeline A — "Standard"**: a Layout Mapper agent reads the whole document, figures out column positions, section headers, and skip ranges, then hands that map to a Data Extractor agent which uses it as a guide. Smart. Thorough. Two AI calls per document.

**Pipeline B — "Single-pass"**: one agent, whole document, one shot. Just extract everything you can see. Fast, simple, no coordination overhead.

**Pipeline C — "Chunked"**: split the PDF into individual pages using a library, run Pipeline B on each page independently, then merge and deduplicate the results. No layout analysis. No cross-page context. Just brute force.

Our hypothesis was that A would win on coverage, B would win on speed, and C would be a reasonable middle ground. We were wrong about coverage.

## The Benchmarks

We ran all three against six real vendor price lists — an 8-page produce sheet, a 6-page dairy list, a 10-page specialty meat catalog, and three versions of the same 2-page broadline vendor sheet. Total across 6 documents:

| Pipeline | Items Extracted | Time |
|---|---|---|
| Standard (layout mapper) | 966 | 637s |
| Single-pass (whole doc) | 507 | 317s |
| **Chunked (per-page)** | **989** | **609s** |

Chunked pulled out **23 more items** than the standard pipeline and ran in nearly the same time. Single-pass was fastest but missed nearly half the items on anything longer than two pages — we already knew why from a previous session, so that wasn't the surprise.

The surprise was that chunked *beat the layout mapper on coverage*.

## Why the Smart Pipeline Lost

The layout mapper is doing real work. On the 10-page specialty meat catalog, it correctly identified 3 columns, 28 section headers, and the specific line ranges to skip. That analysis took a few seconds and cost an extra AI call. But here's what it also did: it introduced a *ceiling*. The extractor agent received a layout map describing the document structure, and when items didn't fit cleanly into that map — odd formatting, section transitions, items at page boundaries — they got dropped.

Chunked has no map to deviate from. Each page is its own fresh prompt. There's no accumulated model of what the document "should" look like. Items near page breaks that would confuse a cross-page layout model just... appear on their respective pages and get extracted normally.

One of the weirder findings came from that 10-page catalog. The layout mapper correctly identified that the document only has three columns: product code, description, and price. No dedicated pack size column. So the 14% pack size extraction rate from the standard pipeline is *accurate* — those items literally embed the size in the product name ("12oz Ribeye"). The single-pass run that claimed 63% pack size extraction was selection bias: it only pulled 110 of 545 items, and happened to grab a section with embedded sizes.

Getting the right answer for the right reasons matters.

## What We Changed

We promoted chunked to the default. The layout mapper is still available as an explicit option — it may prove useful for documents where column structure is genuinely ambiguous and needs pre-analysis. But for the common case of a straightforward price list, the simpler approach does more with less.

The implementation is mechanical: `pdf-lib` splits any PDF into single-page documents, the same single-pass extractor runs on each, SKU + name deduplication merges overlapping results at the boundary. No new prompts, no new models, no new infrastructure.

The lesson isn't that smart pipelines are bad. It's that complexity earns its keep only when you measure it. We assumed the layout mapper was adding value — it was, just not on coverage. Now we know exactly what it costs and what it buys, and we can make the tradeoff deliberately rather than by default.

