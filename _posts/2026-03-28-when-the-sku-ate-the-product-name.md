---
layout: post
title: "When the SKU Ate the Product Name"
date: 2026-03-28 20:12:00 +0000
categories: [debugging]
tags: [pdf-parsing, regex, text-preprocessing, debugging]
excerpt: "A vendor PDF produced garbled product names because the parser couldn't tell where a numeric SKU ended and an all-caps description began — and the fix was a single regex that knew numbers from words."
---

A real vendor PDF came in — a dairy and cheese price list — and about half the parsed product names looked wrong. Not subtly wrong. Aggressively wrong.

The raw bid file contained lines like this:

```
160GRS SALT BUTTER (1 LB) 36 LB 2.8600
273FROZ EGG WHITE 6/5LB CS 62.7800
1346SHRED MOZZARELLA 5LB BAG 15.5200
```

Each line is: numeric SKU, immediately followed by the product description, then pack size, then price. No delimiter between the SKU and the name. Classic PDF copy-paste behavior — the original table had columns, but the text extraction flattened everything into a single stream.

## What the parser saw

We already had a preprocessor handling this kind of concatenation. It detects when PDF columns have been smooshed together and inserts spaces at likely boundaries. Running those lines through it produced:

```
SKU: 160GRS  | Name: SALT BUTTER ()
SKU: 273FROZ | Name: EGG WHITE
SKU: 1346SHRED | Name: MOZZARELLA BAG
```

The SKU field was swallowing the first word of every product name. `160` became `160GRS`. `273` became `273FROZ`. The name fields lost their opening word — the most descriptive part.

## Chasing the rule

The preprocessor works by running each line through a series of transformation rules. Rule 2 was responsible for splitting the SKU prefix from the description:

```javascript
// Rule 2: uppercase/digit prefix followed by a capitalized word
result = result.replace(/^([A-Z0-9]{2,20})([A-Z][a-z])/, '$1 $2')
```

Read that pattern carefully: it fires when an all-caps prefix is followed by **an uppercase letter then a lowercase letter** — a capitalized word like `Prime` or `Fresh`. It was designed for product names that look like `850C109BOPORPrime Beef`.

But this vendor's descriptions are *entirely uppercase*. `GRS SALT BUTTER`. `FROZ EGG WHITE`. `SHRED MOZZARELLA`. There's no lowercase letter anywhere, so the pattern never matched. The SKU and the description stayed glued together.

There was a second problem: the concatenation *detector* — the function that decides whether to run these rules at all — was also blind to this format. It looked for prices jammed against letters (`Something40.35`). But here the price was already space-separated; only the SKU was concatenated. So `detectConcatenatedPdf` returned `false`, and the split rules never even ran.

## The fix

Two small additions:

**Rule 2b** — a new regex that fires specifically when a *purely numeric* prefix (2–6 digits) is followed by two or more uppercase letters:

```javascript
// Rule 2b: numeric SKU prefix concatenated to all-caps description
// "160GRS SALT BUTTER" → "160 GRS SALT BUTTER"
result = result.replace(/^(\d{2,6})([A-Z]{2,})/, '$1 $2')
```

The specificity matters. We only want to split `160GRS`, not something like `4X5LB` (a pack size). Requiring a purely numeric prefix and at least two uppercase letters filters out most false positives.

**Updated detector** — one more condition in the line scanner:

```javascript
// Also flag lines with numeric SKU prefix concatenated to all-caps text
if (/^\d{2,6}[A-Z]{2,}/.test(trimmed)) {
  concatenatedCount++
}
```

This gives the detector visibility into the vendor's format so preprocessing actually activates.

## After the fix

```
SKU: 160  | Name: GRS SALT BUTTER (1 LB)   | Price: $2.86  | Pack: 36 LB
SKU: 273  | Name: FROZ EGG WHITE            | Price: $62.78 | Pack: 6/5 LB
SKU: 1346 | Name: SHRED MOZZARELLA          | Price: $15.52 | Pack: 5 LB
SKU: 8070 | Name: BEL GIOIOSO BURRATA       | Price: $52.40 | Pack: 24/4OZ
```

66 items parsed from the full file, SKUs clean, names intact. All 395 existing tests stayed green.

## The lesson

The original Rule 2 was written for one vendor's format and quietly failed on another. Both are "concatenated PDFs" — but concatenated differently. The detector and the splitter were coupled to the same assumption about what concatenation looks like.

When parsing text from the real world, *every vendor is a special case*. The way to handle that isn't an ever-longer cascade of format-specific hacks — it's making each rule narrow enough that it only fires when it's confident, and explicit enough that you can read exactly what it matches. A regex that says "two-to-six digits followed by two-or-more uppercase letters, at the start of the line" is a rule you can reason about. That specificity is what kept it from breaking everything else.

