---
layout: post
title: "When the Table Extractor Can't Find the Table"
date: 2026-04-26 02:32:23 +0000
categories: [debugging]
tags: [pdf-parsing, llm-vision, tabula, document-extraction]
excerpt: "Tabula's coordinate-based row detection fell apart on a vendor PDF where row height varied — and switching to LLM vision fixed it instantly by letting the model read the page the way a human would."
---


We had a PDF from a large produce distributor that looked perfectly readable to any human — a clean, formatted price list with SKUs, product names, pack sizes, and unit prices marching down the page in tidy rows.

Tabula ate it and produced garbage.

## What Tabula Does (and Why It Usually Works)

Tabula is a well-loved Java library that extracts tables from PDFs by analyzing the spatial coordinates of each character on the page. It looks at where text physically sits, clusters characters into cells by proximity, and reconstructs a CSV. For PDFs with consistent, grid-aligned tables — the kind a spreadsheet generates — it's excellent.

The key assumption is that rows have uniform height. Each product occupies the same vertical slice of the page, so the y-coordinate of each character tells you exactly which row it belongs to.

That assumption didn't hold here.

## The Variable-Height Problem

Vendor PDFs aren't generated from a spreadsheet template. They're often produced by ERP systems that wrap long product names across multiple lines. A "BLUEBERRIES, ORGANIC DRISCOLL 18/6 OZ PINT" entry might use two lines for the name, while "LEMONS 165CT" uses one.

When row height varies, Tabula's spatial clustering breaks. It sees characters at overlapping y-coordinates and has to guess where one row ends and the next begins. In our case:

- SKU numbers were bleeding into the row below them
- Product names were getting truncated at the wrap point
- Multi-line descriptions were split across two "rows," turning one item into two half-items
- Prices from one item were occasionally attributed to a neighboring SKU

The resulting CSV went into our downstream column detector and matched-field extractor, neither of which could compensate for data that was structurally scrambled before they ever touched it.

## What We Tried First

The first instinct was to fix it in post-processing: detect the anomalies, merge rows that looked like they'd been split, re-associate orphaned SKUs. We started sketching out heuristics.

Then we stopped and asked the obvious question: who actually reads this correctly on the first try?

A human does. Or, as it turns out, an LLM with vision.

## The Fix: Route PDFs to Vision Instead

Our Agent Parse pipeline — the one we already used for image uploads — works differently. It converts each PDF page to a rendered image and sends it to the model as a visual. The model sees the page the way a printer would render it, not as a stream of coordinate-tagged characters. It reads columns and rows the way you do: by understanding the layout visually.

The code change was small. PDFs had been hitting the upload endpoint, which tried Tabula first and routed to the spreadsheet-job pipeline on success. We removed that branch and sent PDFs directly to the same per-page vision pipeline we use for photos of printed documents:

```
// Before: PDFs → Tabula → straight-through spreadsheet import
// After:  PDFs → Agent Parse (per-page vision) → same as images
if (ext === 'pdf') {
  await handleMultiImageAgentParse([file])
}
```

No new prompts. No special handling for variable row heights. The model just… handled it.

## Why This Matters

The broader lesson isn't "Tabula is bad" — for well-formed PDFs it remains a solid, fast, free solution. The lesson is about where the brittleness lives.

Coordinate-based extraction is brittle at the layout layer. Any document that doesn't conform to a strict grid will confuse it, and you'll spend significant effort writing compensating logic downstream. Vision-based extraction moves the brittleness to the semantic layer — the model might misread a number or misidentify a column, but it won't structurally scramble the data before you even have a chance to validate it.

For documents from real vendors, with real ERP-generated formatting quirks, structural scrambling is exactly the failure mode that hurts. A semantic misread is fixable. A row that silently merged its SKU into the next item's price column is much harder to catch.

When the document format is the variable, let the model handle the variability.

