---
layout: post
title: "The Search Box That Could Only Find Half the Products"
date: 2026-04-02 00:34:48 +0000
categories: [debugging]
tags: [react, tanstack-query, data-modeling, ux]
excerpt: "Our product match editor had a working search box — it just couldn't find any product that hadn't already been matched to something, which was exactly the case when you needed it most."
---


We got a bug report that felt impossible at first: "I can't find Asparagus in the product search." The product existed in inventory. The search box was right there. But no matter what the user typed, Asparagus never showed up.

## What the Screen Was Supposed to Do

The context: we have a bid analysis screen where the system automatically matches vendor bid items to inventory products. "Case of Asparagus Spears, 30 lb" from one vendor might match to "Asparagus" in your product catalog. When the auto-match is wrong — or missing — there's an editor where you can manually reassign it. You type in the search box, find the right product, click it. Simple.

Except sometimes the right product wasn't there.

## Following the Data

The search wasn't broken. The filtering code was straightforward — `products.filter(p => p.name.toLowerCase().includes(search))` — and it worked exactly as written. The bug was upstream: what was in `products` to begin with.

We traced it back to where the component got its list:

```typescript
const availableProducts = useMemo(() => {
  return data.inventoryGroups.map((g) => ({
    id: g.productId,
    name: g.productName,
    category: g.category,
  }))
}, [data.inventoryGroups])
```

`inventoryGroups` is the analysis result — it's a list of your inventory products that had *at least one bid item match*. If a product existed in your catalog but didn't match anything in the current bid sheets, it wouldn't appear in `inventoryGroups`. And therefore it wouldn't appear in the search.

Asparagus was in inventory. It just hadn't matched anything automatically. So it was invisible in the exact UI designed to help you fix missing matches.

## The Irony

The match editor existed specifically to rescue unmatched items. If the system confidently matched everything, you'd never open it. But the search only showed products that were *already* matched — the products that needed no rescuing. The ones you actually needed to find were the ones that had been left out.

It's a subtle data-modeling mistake. The component was seeded with results from a query scoped to the current analysis session, when what it needed was the full product catalog.

## The Fix

We swapped the data source:

```typescript
// Before: only products that matched something in this session
const availableProducts = data.inventoryGroups.map(...)

// After: every product in inventory
const { data: allProducts = [] } = useProducts()
const availableProducts = useMemo(() => {
  return allProducts.map((p) => ({
    id: p.id,
    name: p.canonicalName,
    category: p.category,
  }))
}, [allProducts])
```

One hook swap. Now the search sees everything. Asparagus shows up.

While we were in there, we noticed the "Current match" info panel showed the match score and tier but not the actual product name. You'd open the editor for a bid item and see "Score 73% — Tier 5 — fuzzy" with no indication of *what* it was matched to. We added the product name to that panel at the same time.

## The Lesson

When a search or filter feels broken, the first question isn't "what's wrong with the query?" — it's "what's in the dataset being queried?" A perfectly correct filter over the wrong input set is still broken.

In this case the input came from a query scoped to the current context (this analysis session) rather than the full domain (the product catalog). Those are easy to conflate when you're building fast, because the session data is right there and it contains product names and IDs that look exactly like what you need. But scope matters. When a UI needs to let users pick from *all* of something, it needs access to *all* of it — not just the subset that happened to be relevant to the last automated step.

