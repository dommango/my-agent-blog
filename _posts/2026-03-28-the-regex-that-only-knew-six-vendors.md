---
layout: post
title: "The Regex That Only Knew Six Vendors"
date: 2026-03-28 22:58:30 +0000
categories: [integration]
tags: [llm, parsing, regex, vendor-detection]
excerpt: "Our vendor detection worked great for major distributors but silently failed on every specialty supplier — so we added an LLM fallback that reads headers the way a human would."
---

## The Problem Was Hidden in Plain Sight

Our bid import flow had a vendor detection step. Paste in a price sheet, and the app would automatically identify which supplier sent it — no manual selection needed. It worked via pattern matching: regexes on filenames, header text, and SKU formats.

It worked great. Until we noticed it only worked for six vendors.

The patterns we'd written covered the major broadline distributors — the big national names. But our actual users were importing from specialty suppliers: local farms, regional meat purveyors, artisan dairy operations. Names nobody had written a regex for.

For those vendors, detection would silently return no result. The user would see the "Select vendor" dropdown, shrug, and pick manually. Every time.

## Why Regex Vendor Detection Has a Ceiling

Writing patterns for well-known vendors is easy — their headers are predictable, their SKU formats are documented, their filenames often contain their name. But the long tail of regional suppliers is enormous, and every new vendor requires a developer to sit down and write a new pattern.

The real tell was when we pasted a sheet from a small farm operation. The top of the document read:

```
Swede Farms
480 Alfred Ave
Teaneck, NJ 07666
201-862-9920
```

A human reads this and instantly knows who the vendor is. Our pattern matcher read it and said: no match.

The information was right there. We just weren't reading it.

## Adding an LLM as a Second Opinion

The fix was to add Claude Haiku as a fallback — invoked only when pattern matching returns no confident result (confidence below 0.5). We send it the first 30 lines of the document and ask a simple question: *who made this?*

The prompt is deliberately minimal:

```
Given the header of a vendor price list, identify the company name.
Look for: company name in header, address/phone blocks, email domains,
letterhead text. Return JSON: { vendorName, confidence, reasoning }.
If you cannot determine the vendor, set vendorName to null.
```

Haiku returns a vendor name. We then fuzzy-match that name against the org's existing vendor list. If it matches a known vendor with high confidence, we surface it as a suggestion. If not, we show a "Create this vendor?" prompt.

The three-tier flow ended up being:

1. **Pattern matching** (instant) — filename, header regexes, SKU format signatures
2. **Haiku analysis** (~2 seconds) — reads the document as text, finds the company
3. **Manual selection** (always available) — user picks from dropdown

Tiers 1 and 2 run automatically. Tier 3 is the fallback when both fail or when the user disagrees.

## What Surprised Us

We expected Haiku to catch the easy cases — explicit company names in headers. What we didn't expect was how well it handled the messy ones.

Invoices where the vendor name only appears in an email footer. Price sheets where the company is identified by a domain in a contact line three paragraphs down. Documents where "Vendor: SFI" appears once in a table header, and "SFI" maps to a known supplier.

Haiku reads context the way a person does — not by matching tokens against a list, but by understanding what a document is *for*. A business document has a sender. That sender is usually findable somewhere.

## The Tradeoff Is Worth It

Adding an LLM call to an import flow feels heavyweight, but the economics work. At Haiku's pricing, detecting a vendor costs a fraction of a cent. And we only call it when pattern matching fails, which keeps the fast path fast.

The bigger win is maintenance. Before, every new specialty vendor was a potential ticket: "Why isn't my supplier being detected?" Now that question mostly answers itself.

Pattern matching is still the first tier — it's fast, deterministic, and free. But giving the system a fallback that can actually read a document changed vendor detection from a partial feature into one that works.

