---
layout: post
title: "The Review Step That Didn't Actually Review Anything"
date: 2026-03-31 12:33:50 +0000
categories: [debugging]
tags: [parsing, data-loss, api-design, typescript]
excerpt: "Our import pipeline had a review screen where users could fix parsed items before saving — but the backend was silently re-parsing the raw text from scratch and throwing all those edits away."
---

Here's a humbling category of bug: the kind where the product *looks* correct, the tests pass, and users don't see an error message. They just quietly get worse results than they should.

We had one of those.

## The Setup

Our import flow for vendor bid sheets has several stages. First, raw text gets extracted from a PDF or spreadsheet. Then a parser runs over it, extracting product names, pack sizes, case prices, and SKUs. Then — critically — a **review screen** shows the user what the parser found, lets them fix misread fields, reject junk rows, and confirm everything looks right before saving.

After review, they click **Import**. Done.

Except it wasn't done. Not really.

## What Was Actually Happening

The review screen was real. Users could edit rows, reject items, sort by confidence. But when they hit Import, the frontend was sending the raw source text to the API — and the backend was *re-parsing it from scratch*.

Every fix the user made? Gone. Every rejected row they'd cleaned out? Back in. The backend had no idea a human had just spent two minutes reviewing anything. It just ran the same parser again on the same raw bytes and saved whatever came out.

The most revealing symptom was in the UI. The file upload badge said **"246 lines extracted"** — which was the raw line count from the PDF, not the number of parsed items. Users would review ~60 clean items, click Import, and the confirmation screen would say "Imported 61 items." Close enough that nobody noticed the discrepancy, but wrong in subtle ways that compounded over time.

## Why It Happened

The bug had a reasonable origin. The parser was built first. The review screen came later, as a UX improvement. When review was added, the import endpoint was never updated to accept the reviewed items — it just kept doing what it always did.

The API contract looked like this:

```ts
// What the endpoint accepted
POST /bids/import
{ vendorId: string, rawText: string }

// What the frontend was sending
POST /bids/import
{ vendorId: string, rawText: string }  // reviewed items: discarded
```

Both sides were technically correct. They just weren't talking about the same thing anymore.

## The Fix

The solution was straightforward once we saw it: make the import endpoint accept an optional `parsedItems` array. When present, use those directly. When absent, fall back to re-parsing — which preserves backward compatibility for any API-only callers.

```ts
// Updated endpoint schema
POST /bids/import
{
  vendorId: string,
  rawText: string,
  parsedItems?: ParsedBidItem[]  // use these when provided
}
```

On the service side, one branch:

```ts
const itemsToSave = body.parsedItems?.length
  ? body.parsedItems           // user-reviewed: use as-is
  : parseBidLines(body.rawText, vendor)  // fallback: re-parse
```

The frontend already had the reviewed items in state. It just needed to include them in the request.

## The Other Fixes That Fell Out

Once we were looking closely at the import flow, a few adjacent issues surfaced:

**"246 lines extracted" was confusing.** It counted raw newlines from the PDF, not parsed items. Changed it to "Ready to parse" — less information, but also less misinformation.

**The review screen sorted by product name by default.** Low-confidence items — the ones most likely to have errors — were scattered throughout. Changed the default sort to confidence ascending, so the worst items rise to the top where users will actually see them.

**"View Items" navigated to an empty dashboard.** After import, the success screen offered two buttons. "View Items" sent users to the analysis dashboard in its final state, which requires a computed analysis to show anything. With no analysis run yet, it showed zeros everywhere. Changed it to navigate to the setup screen instead, with the new bid sheet pre-selected.

## The Lesson

Multi-stage pipelines accumulate drift. Each stage is built at a different time, by someone solving a different problem. A stage added for UX (review) doesn't automatically update the stage it feeds into (import). The gap is usually invisible — no errors, no crashes — until someone notices the output is subtly worse than it should be.

The tell in this case was that the "lines extracted" number and the "items imported" number were almost but not quite the same. A small discrepancy in a pipeline that should be deterministic is worth pulling on.

Silent data loss rarely announces itself. You have to go looking.

