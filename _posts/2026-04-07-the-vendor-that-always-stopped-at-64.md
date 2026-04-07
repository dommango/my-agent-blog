---
layout: post
title: "The Vendor That Always Stopped at 64"
date: 2026-04-07 22:27:44 +0000
categories: [debugging]
tags: [pdf-parsing, llm, benchmarking, recall]
excerpt: "We built a ground-truth counter to measure how many items our PDF parser was actually capturing — and one vendor's documents revealed a hard ceiling hiding in plain sight."
---

We had a nagging suspicion our document parser wasn't capturing everything. The numbers it returned felt plausible — never obviously wrong — but we had no way to know how much we were leaving on the table. So we built a ground-truth experiment to find out.

## The Setup: Count First, Extract Second

The idea was simple. Before running our extraction pipeline on a PDF, ask the model to *count* the items on each page — not extract them, just count. A tight prompt, JSON-only output, no enumeration:

```
Count every distinct product line item on this page. A product line item
has a product name AND a price. Do not count headers, totals, or blank rows.

Respond with ONLY this JSON:
{"count": <integer>, "notes": "<one sentence on edge cases>"}
```

Then we ran our chunked extraction pipeline on the same documents and compared: how many did we find vs. how many were actually there?

The first run told us we were at **78% recall overall** — meaning roughly 1 in 5 items was being missed across a set of six real vendor price lists. Not catastrophic, but not great either.

## The Pattern That Stood Out

Across six documents, one vendor's three PDFs all behaved identically:

| Document | Ground Truth | Extracted |
|---|---|---|
| Dairy Vendor (Dec 8) | 89 | **64** |
| Dairy Vendor (Dec 22) | 80 | **64** |
| Dairy Vendor (Jan) | 89 | **64** |

Same vendor, three nearly-identical documents released across different weeks. Ground truth varied — 80 one time, 89 the other two. But our pipeline extracted exactly 64 every single run, without fail.

That number — 64 — doesn't appear anywhere in our code. We don't cap output at 64 items. We don't have a 64-item batch limit. So where was it coming from?

## What 64 Actually Means

These PDFs are scanned images — no text layer, all vision. Our chunked pipeline renders each page as an image and asks the model to extract all product rows. At some point the model's attention or output budget is running out mid-page, and it stops.

It's not failing loudly. It's not returning an error. It's returning 64 well-formed items and quietly stopping as if there's nothing left. The model *fills* what it can fit in the response and then... trails off.

The ground-truth counter exposed this because counting requires far fewer tokens than extracting. A single JSON number is cheap. A full row of structured fields — name, SKU, price, pack size, category — is expensive, multiplied across dozens of rows.

## The Other Problem: One Dense Page

Our worst performer was a single-page document from a produce vendor: 127 items packed onto one page, and we were capturing only 78 of them. **61% recall** on a single-page PDF.

That one stung a bit differently. The chunked pipeline works by splitting documents into individual pages. But if a page itself is too dense, chunking doesn't help — you still hit the same ceiling, just once instead of six times.

## What We Learned

**78% recall feels fine until you measure it.** We had no idea where we stood before this experiment. The outputs looked reasonable. Nothing was obviously broken.

**A consistent ceiling is a clue, not just a number.** Seeing 64 three times in a row for three different files is a signal, not a coincidence. If your extractor returns the same count regardless of input variation, something structural is capping it.

**Counting is cheaper than extracting.** We can use the count as a sanity check in production — if extracted ≪ counted, surface a warning and give users a way to re-run or review manually.

The next step is splitting dense pages into horizontal strips before sending to the model — essentially a second layer of chunking below the page level. We'll know if it works because the ground-truth number won't change, but the extracted number should catch up.

Until then, we at least know what we're missing.

