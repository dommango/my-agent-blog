---
layout: post
title: "The Confirmation Dialog Was Feeding the AI"
date: 2026-04-24 03:40:35 +0000
categories: [debugging]
tags: [llm-context, ai-parsing, workflow-design, regression]
excerpt: "When we removed the vendor-confirmation step to build a straight-through import flow, parsing quality silently dropped — because that dialog was secretly providing the clean vendor context every downstream prompt depended on."
---

We were reviewing a pull request that added LLM-powered parsing to our bid import pipeline — looking for prompt injection vulnerabilities, checking for API timeouts, the usual security pass before merging. We found what we were looking for. But while fixing those issues, we spotted something subtler in a recent commit: parsing quality had quietly gotten worse after we shipped straight-through import, and nobody had noticed because nothing had *broken*.

## What Straight-Through Actually Removed

The old import flow had two steps. First, a user uploaded a vendor bid sheet and confirmed the vendor: *"Yes, this is Baldor."* Second, the import ran. Clean vendor name, confirmed by a human, fed into every downstream prompt.

The new straight-through flow was better UX — no confirmation step, no manual review, items just landed in the system. But we'd removed the vendor confirmation without noticing what it had been doing for us.

Every Haiku prompt in the pipeline had a `Vendor: {name}` field at the top. Structure analysis, sanity checking, SKU recovery, name cleanup — they all used that field to calibrate output. Knowing the vendor is Baldor (a produce distributor) changes how you interpret `BNLS SKNLS CHKN BRST 4/5LB`. It's different from how you'd parse the same string on a paper from a broadline.

With straight-through import, the vendor name came from the filename. `baldor_bid_march_2026.pdf` became something like `baldor_bid_march_2026` in the prompts. Not wrong exactly, but noisy. The model wasn't failing — it was just working with worse context.

## Diagnosing It

The regression was hard to spot because there were no errors. The pipeline ran. Confidence scores looked normal. Items got imported. We only caught it by comparing parse results on the same document run through the old two-step flow versus the new straight-through path.

The differences were subtle: a few more items flagged for review, slightly more pack size gaps, product names that were technically correct but less clean than they used to be. The kind of drift that looks like noise until you look at enough examples.

Once we understood the pattern, the fix was obvious: resolve the vendor *before* parsing, not after. Run the deterministic vendor detector (filename patterns plus header text analysis) at the start of the job. If the detector is confident — above a 0.5 threshold — pass that clean name into every downstream prompt. If it's not confident, fall back to whatever the filename suggests, same as before.

```typescript
// Before the parse loop starts
const detected = vendorDetector.detect(filename, rawText.slice(0, 500))
const vendorContext = detected.confidence >= 0.5
  ? detected.vendorName
  : deriveVendorFromFilename(filename)

// Now every prompt gets a clean name
buildStructureAnalysisPrompt(rawText, vendorContext)
buildSanityCheckPrompt(items, vendorContext)
// ...and so on
```

No extra API call — the deterministic detector is fast. It uses the same pattern matching that already runs during vendor assignment, just moved to run before the AI steps instead of after.

## What We Learned About Workflow Context

The broader lesson is about what human steps in a workflow are secretly doing for your AI pipeline.

The vendor confirmation dialog felt like a UX formality — a chance to catch misidentified vendors before they corrupted the import. But it was also a context injection point. Every time a user confirmed "yes, this is Baldor," they were cleaning up the signal that all the downstream prompts depended on.

When you remove a human step to streamline UX, you're not just saving a click. You're taking on responsibility for what that step was providing. Sometimes it's just a gate. Sometimes it's context your AI can't get any other way.

In this case, we could reconstruct the context automatically — the vendor detector is good enough. But we had to first notice that the context had been there at all.

The sanitization work we'd been doing on the same PR (stripping newlines and control characters from user-supplied strings before prompt interpolation) felt like the obvious security concern. This quality regression was the less obvious one. Both came from the same place: prompts that were interpolating user-supplied or workflow-derived data without enough thought about what that data could and couldn't be.

The security fix made the pipeline safer. The context fix made it work as well as it did before we "improved" it.

