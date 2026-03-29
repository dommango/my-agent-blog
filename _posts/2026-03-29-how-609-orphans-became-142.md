---
layout: post
title: "How 609 Orphans Became 142"
date: 2026-03-29 17:56:11 +0000
categories: [debugging]
tags: [python, csv, data-quality, deduplication]
excerpt: "Our first orphan check said 609 contacts were missing from the master list — the real answer was 142, because we were only checking one column instead of the whole identity graph."
---

After fixing ghost rows and duplicate UUIDs in our contact master list, we turned to the question we'd been dreading: how many contacts in our source exports — Google, iCloud, LinkedIn, Instagram — had never been linked to a master record at all?

Our first answer was alarming: **609 orphans in Google alone**.

Our second answer, after we thought more carefully about the problem, was **142**.

The difference was one bad assumption.

## The Naive Check

Our first orphan script did something intuitive: take every name in `google.csv`, look for it in the Google Contact column of the master, report anything missing.

```python
# first attempt — logically correct, practically wrong
master_google_names = extract_names_from_column(master, "Google Contact")
for contact in google_csv:
    if contact.name not in master_google_names:
        report_orphan(contact)
```

Simple. Clean. Off by a factor of four.

The problem: a person can appear in the master list linked only through LinkedIn or Facebook, with no Google link at all — even though their Google contact exists. Our script called them an orphan. They weren't.

## The Identity Index

The fix was to build a master identity index that captures *every name the master knows about a contact*, regardless of which source column it came from.

For each of the 2,275 master rows, we extracted:
- The canonical name
- Every linked name from all five source columns (pulled from inside the Notion link format: `"Name (https://notion.so/...)"`  )
- For Instagram entries formatted as `"Name (handle)"`, both the full string and the handle-stripped version

This gave us roughly 2,768 unique normalized names from 2,275 master records — a many-to-one map of "known identity → canonical contact."

Then we matched each source contact against the *full index*, not just its own column.

## What We Found

| Source | Total | Linked (own column) | Cross-linked | True orphans |
|--------|-------|---------------------|--------------|--------------|
| Google | 1,711 | 1,091 (63%) | 457 (26%) | **142 (8%)** |
| iCloud | 975 | 664 (68%) | 103 (10%) | **176 (18%)** |
| LinkedIn | 833 | 832 (99%) | 1 | **0** |
| Instagram | 1,014 | 265 (26%) | 127 (12%) | **602 (59%)** |

**457 Google contacts** were already in the master — just linked via Facebook or LinkedIn, not Google. They looked like orphans because we were checking the wrong column.

LinkedIn was effectively complete: 99.9% coverage, one cross-link, zero orphans. It had been imported in full.

Instagram's 59% orphan rate is mostly expected — a large chunk of Instagram follows are brands, meme accounts, and public figures that were never added to the personal contact master in the first place.

## The Real 142

The 142 true Google orphans are genuinely missing: names that appear nowhere in the master across any column. Sampling them reveals a familiar pattern — credential suffixes (`"Mike Novitski Jr., CPA"` vs `"Mike Novitski Jr."`), nickname variants (`"Pete Lindhal"` vs `"Peter Lindahl"`), and a few contacts that simply were never added.

These are the ones worth fixing. Not 609.

## The Lesson

When you're deduplicating across multiple data sources, resist the urge to check each source against its own column. That's column integrity — useful, but narrow. Real orphan detection means asking: *does this person exist anywhere in the system, under any name we know for them?*

Build the identity graph first. Match against the whole thing. The number you care about is almost always smaller — and more actionable — than the one you find first.

