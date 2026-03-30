---
layout: post
title: "Sending PDF Pages to a Vision Model Instead of Running OCR"
date: 2026-03-30 15:24:15 +0000
categories: [architecture]
tags: [pdf-parsing, vision-models, ai-agents, ocr]
excerpt: "When regex and Haiku couldn't handle unknown vendor layouts, we redesigned the parsing pipeline around Claude's vision — converting PDF pages to images and letting a three-agent system read them the way a human would."
---

Our bid sheet parser had a dirty secret: it worked great for the six vendors we'd seen before. Feed it a PDF from Sysco or US Foods and it'd return a clean list of products, prices, and pack sizes in seconds. Feed it anything else and results varied wildly — garbled names, missed prices, SKUs landing in the wrong column.

The root cause was structural. The pipeline started with text extraction (pdf-parse for digital PDFs, Tesseract for scans), then handed that text to a regex engine that assumed columns were in predictable positions. When a new vendor used a bold header bar instead of plain text, or put item numbers on a second line, or fit three logical columns into two physical ones — the whole thing fell apart.

We'd patched this with Haiku. A structure analysis step would scan the first few hundred characters and produce a "profile" (which columns to expect, whether prices were in the last column or second-to-last). That helped. But it was still working from extracted text, and extracted text loses exactly the spatial information that tells you where one field ends and another begins.

## The Shift: Pages as Images

The insight that unlocked everything: modern vision-capable models don't need extracted text. They can look at a page the same way you do.

So we designed a second parsing mode — "Agent Parse" — that converts each PDF page to an image and sends it directly to a vision model. No text extraction step. No assumptions about column layout. The model sees what a human sees.

The pipeline splits into three agents:

**Agent A — Layout Mapper.** Runs once on page 1 only. Sends the image with a prompt asking it to describe the structure: where are the column headers, what order are the fields in, are there multi-line items, are some rows section dividers rather than products? This produces a layout map that the next agent uses as a guide.

**Agent B — Row Extractor.** Runs per page, carrying the layout map forward. For each page it receives the image plus the layout context from Agent A (and the last section header seen, so category labels on page 1 correctly apply to items on page 3). It returns a raw list of row objects — one per item, with fields filled as best as possible.

**Agent C — Attribute Cleaner.** This one is mostly deterministic. It takes the raw rows and runs them through the same normalizers we already had: pack size pattern matching, unit conversion, keyword-based categorization. Only genuinely ambiguous fields go back to the LLM.

## Why Three Agents Instead of One

We considered a single prompt that did everything — layout detection, extraction, and normalization in one shot. The problem is that vision prompts with lots of instructions and a complex table image tend to produce inconsistent JSON. The model tries to do too many things at once.

Splitting the work keeps each prompt focused. Agent A only needs to describe structure, so its output is short and stable. Agent B only needs to extract rows, so it doesn't have to reason about what "6/5 LB" means as a pack size. Agent C never looks at the image at all — it just cleans up numbers and strings.

It also keeps costs manageable. Agent A runs once per import, not once per page. Agent C is effectively free since it's deterministic code with only occasional LLM fallbacks. Most of the cost is in Agent B, and vision tokens on a modern model run about a tenth of a cent per page for a typical bid sheet.

## What Gets Reused

The existing pipeline didn't go away — it became "Quick Parse," the default for uploads from known vendors with clean text. Agent Parse is the toggle you reach for when Quick Parse returns low-confidence results or when you're onboarding a vendor you've never seen.

Everything downstream is shared: the confidence scoring, the product matching tiers, the human review queue. If Agent Parse extracts a row with low confidence, it flows into the same review interface as always. The output format is identical regardless of which mode extracted it.

## The Lesson

OCR is a lossy transformation. It turns a spatial document into a flat string and then you spend enormous effort trying to recover the structure you just threw away. Vision models skip that step — they work directly from the spatial representation.

That's not always the right trade (vision calls cost more and take longer than regex). But for documents where layout *is* the data, sending the image is often the simplest correct solution.

