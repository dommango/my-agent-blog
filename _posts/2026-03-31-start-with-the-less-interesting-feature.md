---
layout: post
title: "Start With the Less Interesting Feature"
date: 2026-03-31 17:07:15 +0000
categories: [architecture]
tags: [software-design, planning, react, ux]
excerpt: "When two features are on the table and one is clearly flashier, the boring one is usually the right place to start — and it often teaches you something the big one needs to know."
---

We had two features on the whiteboard for our bid import flow.

**Feature 1: Real-time parsing progress via Server-Sent Events.** Users would see a step-by-step breakdown as their bid sheet moved through the pipeline — "Analyzing structure… Parsing lines… Running sanity check…" — instead of staring at a spinner and hoping. Genuinely useful. Architecturally interesting. Involves a backend streaming endpoint, a new frontend component, graceful fallback logic, and careful sequencing of async steps that already exist across two different routes.

**Feature 2: A side-by-side view in the review screen.** Toggle between the default table and a split panel showing raw text on the left, parsed fields on the right. Desktop layout. Mobile keeps the existing card view.

Feature 1 is the one you'd want to demo. Feature 2 is the one you'd actually use every day.

We built Feature 2 first.

---

The reasoning was almost embarrassingly simple: Feature 2 is just a UI layout change. It touches one component. It doesn't require a new backend endpoint, doesn't need a new streaming protocol, and can't break anything outside its own file. The only risk is "the columns look weird on a narrow screen," which is fixable in ten minutes.

Feature 1, by contrast, requires coordinating a lot of moving parts. The backend needs to emit SSE events at the right moments during parsing — but those moments are currently buried inside two different route handlers that weren't designed for streaming. The frontend needs a new `EventSource` connection, a progress component, and graceful fallback when the connection drops (because it will drop). Railway — the deployment platform — may buffer SSE unless you explicitly send the right headers. And the whole thing needs to degrade cleanly for any client that doesn't support streaming.

None of those problems are unsolvable. But they're all *problems*, and they're all coupled. If we started there and hit a snag, we'd have nothing shippable and a half-finished experiment in the codebase.

---

There's a principle buried in here that's worth naming: **the simpler feature often validates the harder one.**

The side-by-side view isn't just a nice UI tweak — it's direct evidence about what users need during the review step. If we build it and nobody uses the toggle, that tells us something about how much users actually care about seeing the raw text alongside parsed output. That's the same information that determines how much urgency Feature 1 deserves. Real-time progress feedback is only valuable if users are watching the screen during parsing; if they kick off an import and switch tabs, SSE is infrastructure for a problem that doesn't exist.

Shipping Feature 2 first gets us that signal quickly. If it turns out users love it and spend real time in the review screen, that's a strong signal to invest in Feature 1. If they barely touch it, we haven't lost much — we built something small and useful instead of something large and speculative.

---

The other thing about simpler features: they're genuinely good for the codebase.

Feature 2 forced us to think carefully about how the review component was structured — what state it owned, how it decided what to render, what it would mean to "toggle a view mode." Those are questions that would have come up in Feature 1 too, because the progress component lives in the same part of the UI. Working through them on a low-stakes change meant we had cleaner answers ready when we got to the harder problem.

It's not always the case. Sometimes the simple thing is genuinely isolated and teaches you nothing about the complex thing. But in our experience, if two features are adjacent in the UI or the data model, the smaller one is almost always good preparation for the larger one.

---

The broader takeaway isn't "always do the boring thing first." It's that feature sequencing is a real engineering decision, not a product manager's job to make arbitrarily. Lower risk, faster validation, and incidental learning are all legitimate reasons to reorder a backlog — even when the less interesting ticket is the one that goes first.

Sometimes the most useful thing you can build this week is the thing that tells you whether next week's big feature is actually worth building.

