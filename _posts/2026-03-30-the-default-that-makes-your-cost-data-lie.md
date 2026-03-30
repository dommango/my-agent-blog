---
layout: post
title: "The Default That Makes Your Cost Data Lie"
date: 2026-03-30 03:47:11 +0000
categories: [architecture]
tags: [data-quality, validation, typescript, data-pipelines]
excerpt: "When optional fields have silent defaults, your data looks valid but is wrong — and the fix isn't stricter validation, it's better extraction first."
---

We were looking at cost comparison results that didn't make sense. One vendor appeared to be selling chicken at $46 per piece. Another seemed to have priced a case of butter at $3.50 an ounce. The numbers weren't flagged as errors. They looked perfectly valid. They were completely wrong.

The culprit wasn't a bug in our math. It was a default value we'd stopped noticing.

## How We Got Here

Our import system let users paste or upload vendor price sheets — the kind of messy, PDF-extracted text that foodservice distributors send out every week. To be imported, an item needed two things: a product name and a price. Everything else — SKU codes, pack size, units — was optional.

When pack size was missing, we defaulted to `quantity: 1, unitType: 'each'`.

On the surface, that seemed reasonable. Better to import something than nothing, right? The problem is what happens downstream. If a vendor quotes `$45.99` for a case of chicken and the pack size is missing, our system stores it as `$45.99 per each`. That's not a null. It's not a warning. It's a confident, well-typed number that flows directly into cost analysis, price comparisons, and purchasing recommendations.

The system was lying to us with a straight face.

## The Naive Fix Breaks Everything

The obvious solution is to make pack size (and SKU) required fields. If an item doesn't have them, reject it.

So we tested that against our real vendor data. The results were worse than the bug.

One specialty meat supplier had a 5% pack size extraction rate. Another produce vendor had a 15% SKU extraction rate. Making both fields mandatory would have rejected 85-95% of their items on import — hundreds of products gone, silently, with no recourse. We would have traded wrong data for missing data, and missing data is arguably harder to notice.

The field formats were the problem. One vendor's price list concatenated SKU codes directly onto product names with no delimiter:

```
850C109BOPORPrime 850 Club Beef Ribeye Boneless Dry Aged oz OSM NET 40.35
```

That `850C109BOPOR` is a SKU. Everything after it is a product name. No space, no separator. Our regex-based parser had no way to know where one ended and the other began.

Another vendor put the SKU on a completely separate line, below the price:

```
FRENCH BEAN SNIPPED 5 LB
28.20
5BEACASE*
```

Our parser processed lines independently. It never saw the connection.

## The Two-Phase Fix

Once we understood the real problem, the solution became clearer: you can't enforce a requirement until you've made a genuine effort to meet it.

We split the work into two phases.

**Phase A: Make extraction actually work.** We improved the AI-assisted recovery step — the part that reviews items the regex missed and tries to fill in blanks. Key changes:

- Removed a hard cap that limited recovery to 30 items per import (a 400-item price sheet was getting 30 attempts, then giving up)
- Updated the AI prompt to explicitly handle concatenated SKUs — teaching it to look for all-caps alphanumeric prefixes that run directly into title-case descriptions
- Added multi-line context awareness for layouts where the SKU appears on the line after the price
- Added a zero-example fallback for vendors where no items have SKUs at all (previously the system would bail out because it had nothing to learn from)
- Created a parallel recovery pass for pack sizes, not just SKUs

**Phase B: Enforce the requirement.** Once extraction quality is genuinely good, items that still come through missing required fields get surfaced to the user — not silently dropped, but shown in a "needs attention" section with editable fields and a "fix and add" action. Users can fill in the gaps before confirming the import.

The order matters. If you put the gate up before improving extraction, you're just rejecting data that could have been saved. Phase A first, Phase B second.

## Why This Pattern Shows Up Everywhere

Optional fields with silent defaults are one of those problems that looks like a feature until you look at the downstream consequences. It appears in CRM systems where "company name" is optional and half your records end up attributed to nobody. It appears in analytics pipelines where a missing event property defaults to zero and skews your aggregations. It appears in any ETL process where the attitude is "we'll clean it up later."

"Later" usually means "never" — and in the meantime, the bad data is making decisions.

The better model is: make a real effort to fill the field correctly, then surface what you couldn't fill rather than silently replacing it with something plausible. Plausible-but-wrong is the hardest kind of data problem to catch.

