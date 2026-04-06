---
layout: post
title: "The Pipeline That Forgot About Photos"
date: 2026-04-06 03:50:13 +0000
categories: [architecture]
tags: [document-parsing, llm, vision, refactoring]
excerpt: "We built a vision-powered PDF parser that could read any vendor layout — then realized photos taken on a phone were getting routed around it entirely, straight to the weak OCR fallback."
---

## The Pipeline That Forgot About Photos

We'd spent weeks building a three-agent vision pipeline for parsing vendor bid sheets. Feed it a PDF, and it would convert each page to an image, send it to a vision model, and extract structured product data — names, prices, pack sizes — regardless of how unusual the layout was.

It worked beautifully. Then someone uploaded a photo they'd taken of a printed bid sheet.

The system processed it just fine. But when we looked closely at the code, we realized the photo had bypassed the vision pipeline entirely. It went to Tesseract — our OCR fallback — extracted raw text, and then hit the older regex-based parser. The output looked plausible but was missing everything the vision pipeline had learned to do: intelligent section handling, layout-aware extraction, abbreviation expansion.

We'd built a first-class pipeline for PDFs and quietly left images as second-class citizens.

## Why It Happened

The routing logic was straightforward to the point of blindness:

```
if (file.mimetype === 'application/pdf') → vision pipeline
else → OCR fallback
```

This made sense when we first built the vision pipeline — PDFs were the primary format. But as users started photographing bid sheets on their phones, the split became a problem. A scanned page inside a PDF and a JPEG of the same page contain identical information, but they were taking completely different code paths with different quality levels.

The irony: a vision model handles images *natively*. It doesn't need OCR as an intermediate step. We were routing image files away from the tool best suited to read them.

## The Fix: Unify the Routing

The change was conceptually simple — route images to the vision pipeline just like PDFs. Each uploaded image becomes one "page," and the agent system processes it with the same layout-aware approach it uses for PDF pages.

Multi-page bid sheets (users photographing each page separately) also needed to work as a single import. The vision pipeline already handled multi-page PDFs by looping through pages with section-header continuity — images just slot into the same loop.

## The Other Gap: Name Cleanup Parity

While we were in there, we noticed a second inconsistency. The quick-parse flow (for pasted text) had a step 6: send product names to the model for abbreviation expansion. "BNLS SKNLS CHKN BRST" would come out as "Chicken Breast, Boneless Skinless."

The vision pipeline had no equivalent. It extracted names but never cleaned them.

The fix was to extract the name cleanup into a shared function:

```typescript
// Used by both Quick Parse (Step 6) and Agent Parse (after Agent C)
export async function aiCleanProductNames(
  items: ParsedBidItem[],
  vendorName?: string
): Promise<ParsedBidItem[]> {
  if (!aiProvider.enabled) return items

  const normalized = await aiNormalizeProductNames(aiProvider, nameItems, ...)
  return items.map(item => {
    const norm = normalized.get(item.cleanName)
    return norm?.normalizedName ? { ...item, cleanName: norm.normalizedName } : item
  })
}
```

Two lines added to the Agent Parse route, one shared function, and both paths now produce the same name quality.

## The Lesson

When you build a better version of something, it's easy to wire it only into the path you're actively improving. The old path keeps working, nobody complains, and the gap quietly widens.

In our case: the vision pipeline got smarter over months, while image uploads kept routing to the 2019-era OCR approach. The fix took an afternoon. Finding the gap took a lot longer.

The pattern worth remembering: every time you improve a pipeline, ask which *other* entry points feed into it — and whether they're all getting the same treatment.

