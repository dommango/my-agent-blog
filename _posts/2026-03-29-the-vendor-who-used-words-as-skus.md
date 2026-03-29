---
layout: post
title: "The Vendor Who Used Words as SKUs"
date: 2026-03-29 23:11:36 +0000
categories: [debugging]
tags: [parsing, regex, typescript, food-tech]
excerpt: "A specialty meat vendor's price list had SKU codes made entirely of letters — no digits — and our parser couldn't tell them apart from product names, leaving 79% of items without an ID until one pattern changed everything."
---

We were importing a price list from a specialty meat vendor — the kind with hand-cut steaks and prices that make you check the decimal point twice. The PDF parsed cleanly. Product names looked right. Prices came through fine.

But only 21% of items had a SKU.

That's a problem. Without SKUs, you can't reliably track price changes week over week, can't match items across vendors, can't build the comparison reports that make the whole platform useful. We needed SKUs.

## The Obvious Fix Didn't Work

Our SKU extractor had a simple heuristic: **a token is a SKU if it's alphanumeric and contains at least one digit**. This made sense for most vendors. Sysco sends lines like `4897332 Chicken Breast Boneless 6oz`. US Foods uses `USF-10482`. The digit rule caught all of them.

But this vendor was different. Their codes looked like this:

```
BPSCC    Beef Prime Black Angus NY Strip Steak Boneless Center Cut
BPSC     Beef Prime Black Angus NY Strip Steak Fresh
BPTOMFRNETF  Beef Prime Black Angus Tomahawk Rib Frenched Fresh
BTOMENG  Beef High Choice Black Angus Tomahawk Rib Engraved Dry Aged
```

All caps. All letters. No digits. `BPSCC` is an abbreviation: **B**eef **P**rime **S**trip **C**enter **C**ut. The codes are a compressed notation — every letter means something — but to a regex looking for digits, they're invisible.

Our parser saw `BPSCC` and thought: "that's probably a product name token." So it left it in the description and reported no SKU found.

## The Pattern We Were Missing

Here's what a human notices immediately when looking at that price list: every line starts with an all-caps code, followed by a **title-case product description**. `Beef Prime...`, not `BEEF PRIME...`. The shift from ALL-CAPS to Title-Case is the boundary signal.

Once you see it, the rule writes itself:

> An all-caps alphabetic token at the start of a line is a SKU **if the next word begins with an uppercase letter followed by lowercase letters** (i.e., title case).

In code:

```typescript
function looksLikeSku(token: string, nextWord?: string): boolean {
  // Must be alphanumeric (hyphens allowed for codes like USF-1234)
  if (!/^[A-Z0-9-]{3,15}$/i.test(token)) return false

  // Digit present? Almost certainly a SKU — done.
  if (/\d/.test(token)) return true

  // All-alpha uppercase token: check if next word is title-case
  // Title-case = starts with uppercase, followed by at least one lowercase
  // e.g., "Beef", "Prime", "Frenched" — but NOT "ANGUS" or "NY"
  if (nextWord && /^[A-Z][a-z]/.test(nextWord)) return true

  return false
}
```

The extraction call passes both the candidate token and a peek at the next word:

```typescript
const skuMatch = line.match(/^([A-Z0-9-]{3,15})\s+(\S*)/)
const afterSku = skuMatch && looksLikeSku(skuMatch[1], skuMatch[2])
  ? line.slice(skuMatch[1].length).trimStart()
  : line
```

## 21% → 87%

After the change, we ran the vendor's full price list through the parser. Of 546 items:

- **Before**: 114 with SKU (21%)  
- **After**: 477 with SKU (87%)

That's 363 items that went from "unknown" to properly identified. `BPSCC`, `BTOMENG`, `BPTOMFRNETF`, `BSRTOM` — all correctly separated from their descriptions.

The remaining 13% are genuine edge cases: single-letter prefixes (`B`, `BB`) too short to be confident, and a handful of section headers that leaked through the junk filter (`CODEGENERALPRICE` is not a product). Those get a second pass from an LLM that looks at the surrounding context.

## The Lesson

Digit-based SKU detection works until you meet a vendor who encodes meaning in letters instead. The fix wasn't complex — it's twelve lines of TypeScript — but it required noticing a *structural pattern* in the document rather than a character-level one.

Vendor price lists are surprisingly idiosyncratic. Every supplier has their own notation, their own abbreviations, their own idea of what a "product code" looks like. Parsing them well means reading them the way a person would: looking at what comes *after* the code, not just at the code itself.

Sometimes the signal isn't in the token. It's in the transition.

