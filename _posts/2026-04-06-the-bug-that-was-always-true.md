---
layout: post
title: "The Bug That Was Always True"
date: 2026-04-06 22:16:31 +0000
categories: [debugging]
tags: [javascript, debugging, string-methods, data-validation]
excerpt: "A one-character typo in our anomaly detector — an empty string instead of a dollar sign — made every pack size look suspicious, and it took us a while to notice because the code looked completely reasonable at a glance."
---


There's a special category of bug where the code looks right, the tests pass, and the feature appears to work — but silently, something is always wrong.

We found one of those last week in our anomaly detector.

## The Setup

Our parsing pipeline processes vendor bid sheets — lines like `"CHKN BRST 6/5LB $42.50"` — and extracts structured fields: product name, pack size, price. After parsing, an anomaly detector scans the results for suspicious values before anything gets saved.

One check was meant to flag pack sizes that looked like they might have a price accidentally baked in — things like `"5LB$"` or `"12CT$"`. The idea: if a pack size string ends with a dollar sign, something probably went wrong in the extraction.

## The Bug

The check looked like this:

```typescript
if (item.packSize.endsWith("")) {
  flagAnomaly(item, "pack size contains price symbol")
}
```

See it? `endsWith("")`.

In JavaScript, `"".endsWith("")` is `true`. `"hello".endsWith("")` is `true`. *Every string ends with the empty string.* It's how the spec works — an empty suffix matches by definition.

So our anomaly detector was flagging every single item with a pack size. Not just the suspicious ones. All of them.

## Why It Hid So Well

The reason this survived for a while: the downstream behavior was plausible. Flagged items got routed to a human review queue. We expected *some* items to land there. The queue never looked empty in a suspicious way, because plenty of items have genuinely low confidence and belong there anyway.

What we didn't notice was that the "pack size contains price symbol" reason code was appearing on items with perfectly normal pack sizes like `"4/5LB"` and `"12CT"`. There was no alarm — just a queue that was a bit fuller than it should have been.

The fix was a single character:

```typescript
if (item.packSize.endsWith("$")) {
  flagAnomaly(item, "pack size contains price symbol")
}
```

## The Lesson

`endsWith("")` doesn't throw. It doesn't warn. It just returns `true`, politely, every time. Same goes for `startsWith("")` and `includes("")`. They're valid calls — just almost never what you meant.

The deeper issue is that this class of bug — a condition that's *always* true or always false — is hard to catch with happy-path tests. Our test for this check verified that a pack size like `"5LB$"` got flagged. It did. The test passed. We never wrote a test asserting that `"5LB"` *didn't* get flagged, which would have caught it immediately.

If a condition never flips, it's not doing anything. That's worth checking — especially in validation and anomaly detection code, where the whole job is to distinguish the normal from the weird.

