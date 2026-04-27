---
layout: post
title: "The Column That Needed a Better Name"
date: 2026-04-27 03:09:51 +0000
categories: [architecture]
tags: [react, ux, data-modeling, tanstack-query]
excerpt: "When an auto-matching system has three meaningfully different states but the UI labels all of them \"Match,\" users can't tell what's confirmed, what's a suggestion, and what needs their attention."
---


We have a feature that automatically matches vendor products to items in your inventory catalog. When you upload a price list from a distributor, the system tries to figure out that "BNLS SKNLS CHKN BRST 4OZ" is the same thing as "Chicken Breast, Boneless Skinless" in your catalog. It does this using a mix of fuzzy text matching, SKU lookups, and vector similarity search.

The matching mostly works. But during a recent review session, a simple question stopped us cold: *"What does 'Match' actually mean?"*

## Three States, One Label

The problem was that "Match" — our column header in the Vendor Bids view — covered three meaningfully different situations:

- **Unmatched**: The system couldn't find anything in the catalog that plausibly corresponds to this vendor product. No link exists at all.
- **Unverified**: The auto-matcher found a candidate and linked the two products — but no human has confirmed the link yet. It's a *suggestion*, not a fact.
- **Verified**: A human clicked "Verify," confirming that yes, the vendor's product really does map to that catalog item. This is now a trusted, locked connection that trains future imports from the same vendor.

These are fundamentally different states with different implications. An unverified match affects cost calculations but might be wrong. An unmatched item is invisible to your recipe costing until someone handles it. A verified match is as reliable as something a person checked.

The column header said "Match." The badge showed green, amber, or red. But there was no explanation of what those colors meant, and anyone new to the system had to guess.

The fix was straightforward: rename the column to **Inventory Match** and add an info tooltip that explains all three states in plain language. Renaming forced the question — "inventory match" implies "matched to something in your inventory catalog" — and the tooltip closes the loop with one sentence per state.

## The Canonical Name Was in the Wrong Column

While we were in there, we noticed something else: when the auto-matcher found a candidate, the canonical product name (the one from your catalog) was displayed as small subtext *under the vendor product name*, in the Product Name column. So it looked like:

```
BNLS SKNLS CHKN BRST 4OZ
→ Chicken Breast, Boneless Skinless
```

This placed the answer in the question column. The canonical name is the *result* of the match — it belongs under the Inventory Match badge, not next to the raw vendor string. We moved it, and suddenly both columns read correctly: one tells you what the vendor called it, the other tells you what the system thinks it is.

## The Raw Text Toggle

The parser records the original extracted line from the PDF or CSV — the exact bytes before any cleanup. We were showing this as a monospaced footnote under every product name, which was useful for debugging a bad parse but noisy for normal review.

```
BNLS SKNLS CHKN BRST 4OZ
[raw: "  CHK-BRST-4 BNLS SKNLS CHKN BRST  /4OZ    @14.52  "]
```

The raw line is valuable maybe 5% of the time — when you're trying to understand why the parser got a price wrong, or why it split a row incorrectly. The other 95% of the time it's clutter that makes the table harder to read.

We added a "Show raw text" toggle in the toolbar, defaulting to off. If you're debugging a misparse, flip it on and you immediately see the pre-cleanup source for every row. Otherwise, the table stays readable.

## The Timestamp That Was Missing Half Its Data

Last thing: the "By Upload" tab lists each bid sheet in reverse chronological order, and each row showed the upload date as "Apr 26" — just the date, no time.

This is fine when you upload one sheet per day. But in practice you might upload three iterations of the same vendor's list in an afternoon as you troubleshoot a parse issue. Three rows all labeled "Apr 26" with no way to tell which came first.

The fix was one line: switch from `toLocaleDateString` to a combined format that includes the time — "Apr 26, 2:43 PM." Same info, just enough precision to be useful when you're working through multiple uploads in a session.

## Why This Kind of Cleanup Matters

None of these changes touched the underlying matching logic. The system was doing the right thing all along — it's just that the UI wasn't communicating what the system knew.

Auto-matching systems are probabilistic by nature. They generate candidates, assign confidence scores, and wait for human confirmation. If the UI flattens all of that into a single "Match" badge, you lose the signal. Users either over-trust the auto-matches (treating suggestions as facts) or under-trust them (re-verifying things that are already locked in).

Surfacing the three states clearly — unmatched, unverified, verified — gives users the information they need to make good decisions without having to understand how the matching pipeline works internally.

Sometimes the right fix isn't smarter code. It's better labels.

