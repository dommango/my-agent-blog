---
layout: post
title: "What If the AI Could Improve Its Own Code?"
date: 2026-04-04 16:54:00 +0000
categories: [architecture]
tags: [agents, benchmarking, TypeScript, docker]
excerpt: "We designed a benchmark loop where a meta-agent autonomously edits our parsing and matching code, runs tests, scores the results, and keeps or discards its own changes — with no human in the loop."
---


Our bid parsing system had hit 89.8% matching accuracy and we knew exactly what was failing. Domain synonyms we hadn't taught it yet — "belly strips" should map to bacon, "squid" to calamari, "prawns" to shrimp. Extreme abbreviations like `CHKN BRST` pointing at the wrong product. Confidence thresholds that were either too tight or too loose depending on the vendor.

We could fix these by hand. We'd done it before. But there's a ceiling to how fast a human can iterate on abbreviation dictionaries and fuzzy scorer weights, especially when every change might help one vendor and hurt another.

So we asked: what if we gave the AI access to its own source code and a scoreboard?

## The Idea: Autonomous Hill-Climbing

We came across a framework called AutoAgent — a meta-agent loop for autonomous agent engineering. The concept is elegant:

1. Define benchmark tasks with verifiable, deterministic scoring
2. An inner agent (running inside Docker) attempts to solve them
3. A meta-agent reads the failures and edits the inner agent's code
4. Rebuild, re-run, compare scores — keep the change if it helps, discard if it doesn't
5. Repeat indefinitely

In the original setup, the agent being improved is a general-purpose coding bot solving shell tasks. But the same loop can target anything with a measurable score. Including a restaurant bid parsing pipeline.

## What We'd Put in the Loop

Our parsing system has two improvement targets:

**The matching pipeline** — 70 ground-truth fixtures, four difficulty levels (easy through extreme). Correct match rate is already measured by our benchmark tests. Every time the inner agent edits an abbreviation map, fuzzy weight, or confidence threshold and re-runs the benchmark, it gets back a score from 0.0 to 1.0. No ambiguity about whether the change helped.

**The parsing pipeline** — Real vendor text extracts from four vendors in our test data. For each vendor, we annotate ground truth: what should each line's clean name, price, pack size, category, and SKU be. The scoring compares the parser's output field-by-field. A partial improvement (getting the price right but not the pack size) still moves the score — there's a gradient to follow, not just pass/fail.

The inner agent gets the full TypeScript source: parsers, normalizers, fuzzy matchers, AI prompt files. It also gets a real Anthropic API key so it can actually run the AI steps and see how prompt changes affect results. It edits the source, runs the tests, reads the output.

## The Container Design

The tricky design problem was isolation. We didn't want an autonomous agent editing our production parsing code directly. So the Docker container gets its own copy — a snapshot of the relevant packages, stripped of the API server and database layer, but with the full parsing and matching logic intact.

```
container/sousiq-src/
├── bid-parser.ts
├── matching/
│   ├── matching-pipeline.ts
│   └── fuzzy-matcher.ts
├── normalizers/
├── patterns/
├── ai/          ← full prompt files, editable
└── __tests__/   ← benchmark tests, read-only ground truth
```

The agent edits the source files. The test runner executes them. The score goes to `/logs/reward.txt`. The meta-agent reads scores across all seven tasks (four parsing, three matching), compares against the previous run, and decides whether to keep or discard the changes.

When something genuinely improves — when the extreme-difficulty match rate goes from 60% to 70% — we diff the modified files against our production copies and cherry-pick the improvement back in. The loop finds it; a human reviews it.

## The Overfitting Guard

The thing we thought hardest about: how do you prevent the agent from just memorizing the ground truth?

There's a simple heuristic in the meta-agent's directive: *if this vendor disappeared, would this improvement still help?* Vendor-specific hacks don't pass the test. But teaching the system that "prawns" is an alias for shrimp, or that pack sizes in parentheses should be stripped from product names, generalizes across every vendor we'll ever see.

The benchmark fixture set is already built to test generalization — 70 items from four different vendors, none of which have identical formatting. Overfitting to one vendor's patterns will hurt another's score.

## Why We Spent Time Designing Before Building

It would've been tempting to just start wiring things together. But there were real architectural choices with lasting consequences:

- Include the AI provider (so prompt changes have real effect) vs. mock it (cheaper but unable to improve prompts)
- Run in Docker (proper isolation) vs. directly against source (faster but risky)
- Use continuous scoring (gradient to follow) vs. binary pass/fail (easy to implement but noisy signal)

Getting these wrong would mean running the loop for hours and getting no useful signal. The design session paid for itself in avoided dead ends.

The loop is ready to run. The benchmark baseline is set. Now we let it work.

