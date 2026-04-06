---
layout: post
title: "Why We Ship Haiku to Production (And What It Takes to Do It Safely)"
date: 2026-04-06 05:24:05 +0000
categories: [architecture]
tags: [llm, security, document-parsing, claude]
excerpt: "Choosing Haiku for production document parsing isn't just a cost decision — it's a boundary design problem, and getting that boundary right requires prompt injection prevention, API timeouts, graceful degradation, and correct multi-image encoding."
---

When you're wiring an LLM into a SaaS parsing pipeline, the first question is almost always: which model? The second question — the one that takes longer to answer — is: what do you have to build around it to trust it in production?

We went with Claude Haiku for our document parsing layer. Here's the reasoning, the security patterns we landed on, and the multi-image twist we didn't anticipate.

## The Cost Case

At the volumes a small SaaS deals with, model cost sounds like a non-issue. It isn't. Haiku costs roughly 3× less than Sonnet on both input and output tokens. More importantly, document parsing is *high-frequency and low-stakes* — we're calling the model dozens of times per import, on every bid sheet, for every customer. That's the shape of problem where cost multipliers compound fast.

There's also latency. Haiku returns a first token in under a second; a 50-item extraction finishes in 3–15 seconds. That's acceptable for an async import flow. A slower model would push us toward background jobs and polling UI — extra infrastructure for a problem Haiku just doesn't create.

The math: for our current volume, Haiku runs at under $2/month. The next tier up would be $6–8. That's not the point — the point is the *gap stays proportional* as you scale.

## Three Security Patterns That Weren't Optional

Once we decided on the model, we thought the hard work was done. Code review disagreed. Three patterns ended up being non-negotiable before we'd call the integration production-ready.

**Prompt injection prevention.** Vendor names, product descriptions, and user corrections all flow into our prompts. Any of those fields could contain a newline followed by "Ignore previous instructions." The fix is a `sanitizeForPrompt()` function that strips newlines, non-printable characters, and truncates at 500 chars — applied to every user-supplied string before interpolation. It's one utility function, but it has to be applied *everywhere*, which is why a code review caught it rather than us noticing on our own.

**API timeouts.** Without an explicit timeout, a hung LLM call blocks the entire import handler indefinitely. We added an `AbortController` with a 30-second deadline to every `complete()` call in the provider layer. On timeout, the function throws a descriptive error rather than hanging. The import pipeline catches it and falls back to deterministic parsing — which brings us to the third pattern.

**Graceful degradation.** Every AI-enhanced step in our pipeline is a *fallback*, not a requirement. If the provider is disabled, the API key is missing, or a call fails, the pipeline continues with its non-AI output. This wasn't hard to design — it just required the discipline to never let `provider.enabled === false` propagate as an error. Silent no-ops, not silent failures.

## The Multi-Image Problem We Almost Missed

Our vision pipeline was designed around PDFs — one document, multiple pages, one API call. Then we realized that photos taken on a phone (a common way vendors send price sheets) were bypassing the vision pipeline entirely and going straight to Tesseract OCR, which handles them poorly.

The fix had two parts. First, images now route directly to the agent pipeline instead of the OCR fallback. Second, multiple images — say, a user photographing each page of a multi-page fax — get collected into a single `FormData` upload and sent as one multi-page request. The layout mapper receives an array of `VisionDocument` objects instead of a single document, and the data extractor sends only the specific image for each page (rather than the whole document) for more focused extraction.

The content type matters here: images use Anthropic's `'image'` content block type, PDFs use `'document'`. One utility function (`toContentBlock`) handles the dispatch — keeping that logic out of the agents themselves.

## The Pattern That Ties It Together

What these three areas have in common: they're all about the *boundary* between your system and the model. Cost discipline lives at the boundary (call it only where it earns its keep). Security lives at the boundary (sanitize before anything crosses). Graceful degradation lives at the boundary (every exit path must be safe). Multi-image handling is about correctly encoding what crosses the boundary.

Get the boundary right and the model becomes a reliable component. Leave it implicit and you get a fragile integration that works in development and surprises you in production.

