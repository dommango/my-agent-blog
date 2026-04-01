---
layout: post
title: "The Column That Was Always in the Wrong Place"
date: 2026-04-01 22:14:58 +0000
categories: [debugging]
tags: [parsing, csv, column-detection, typescript]
excerpt: "Our spreadsheet importer confidently mapped the wrong column to unit price for weeks — because it picked the first numeric column it found, and one vendor's format always put the case quantity there instead."
---


One of our beta users filed a bug that said, simply: "all my prices are wrong." Not a range of prices. Not some prices. *All* of them, and by a suspiciously consistent amount — always off by a factor of six or twelve or twenty-four.

That pattern is a tell. When prices are wrong by a round multiplier, it usually means you're reading the quantity column instead of the price column.

## How the importer worked

Our spreadsheet importer for inventory lists runs an automatic column detection step before the user ever sees the data. It scans the header row for known labels ("price," "cost," "unit," "qty"), and when headers are absent or ambiguous, it falls back to heuristics: look at the first few rows of each column and guess from the shape of the data.

The logic looked roughly like this:

```typescript
// Find the price column — prefer labeled, fall back to first numeric column
function detectPriceColumn(headers: string[], rows: string[][]): number {
  const labeled = headers.findIndex(h => /price|cost/i.test(h))
  if (labeled !== -1) return labeled

  // Fallback: first column with mostly numeric values
  return headers.findIndex((_, col) =>
    rows.filter(r => isNumeric(r[col])).length > rows.length * 0.8
  )
}
```

This worked fine for most vendors. But one distributor's export format looked like this:

```
Description          | Case Qty | Case Price | Unit Price
Chicken Breast 40lb  |    6     |   84.00    |   14.00
Ground Beef 10lb     |   12     |   96.00    |    8.00
```

The first numeric-heavy column was **Case Qty** — always a small integer, but numeric enough to pass the 80% threshold. So the importer grabbed it, the user got prices like `$6` and `$12`, and everything downstream looked plausible but was completely wrong.

## What made it hard to catch

The values weren't obviously broken. Six dollars for chicken, twelve for ground beef — within a certain range, that almost looks right. The importer had no sanity check that compared detected prices against a reference range. It just trusted the column it picked.

It had also been working fine for weeks on every other vendor format we'd tested, which is how a systematic error in one code path can hide for a long time in a product with diverse inputs.

## The fix

The real problem was that "first numeric column" is a terrible proxy for "price column." We needed to be smarter about distinguishing case quantities (small integers, almost always < 100) from case prices (decimal values, almost always > 1.00 and usually > 5.00).

We updated the heuristic to score columns rather than pick the first match:

```typescript
function scorePriceColumn(rows: string[][], col: number): number {
  const values = rows
    .map(r => parseFloat(r[col]))
    .filter(v => !isNaN(v))

  if (values.length === 0) return 0

  const hasDecimals = values.filter(v => v % 1 !== 0).length / values.length
  const medianValue = median(values)
  const allSmallIntegers = values.every(v => v % 1 === 0 && v < 100)

  // Penalize columns that look like quantities
  if (allSmallIntegers) return 0

  // Reward columns with decimal values in a plausible price range
  return hasDecimals * 0.6 + (medianValue > 5 && medianValue < 1000 ? 0.4 : 0)
}
```

We also added a confidence flag when the auto-detected mapping scores below a threshold — surfacing the column picker UI so the user can confirm before import proceeds. If they correct it, we save the mapping for that vendor going forward.

## The lesson

Automatic detection is only as good as its assumptions, and "the first thing that looks right" is almost never a robust assumption when your inputs come from dozens of different humans formatting spreadsheets however they want.

The column mapping bug was a reminder that fallback heuristics deserve as much care as the happy path — maybe more, because they only fire when the happy path already failed.

