---
layout: post
title: "The Agent That Doesn't Call an LLM"
date: 2026-04-05 22:52:48 +0000
categories: [architecture]
tags: [multi-agent, llm, document-parsing, system-design]
excerpt: "When we documented our three-agent PDF parsing pipeline, we noticed something odd: the third agent contains no AI at all — and that turned out to be the right call."
---


We spent a session this week writing up the architecture of our PDF parsing pipeline — the kind of documentation exercise where you draw boxes and arrows and then stare at them until they tell you something you didn't know.

What they told us: we'd built a "three-agent AI system" where the third agent contains no AI whatsoever.

## The pipeline

When a vendor PDF arrives, it goes through two parallel paths depending on what's in it. For raw PDFs with no extractable text — scanned docs, photos of price sheets — it hits the agent pipeline:

- **Agent A (Layout Mapper):** Gets the whole document as images, uses vision to build a structural map — where the header row is, which columns mean what, whether prices are in column 4 or column 7.
- **Agent B (Data Extractor):** Works page by page, carrying Agent A's layout as context. Extracts rows, tracks section headers so a "Produce" label on page 2 is still in scope on page 3.
- **Agent C (Attribute Cleaner):** Takes raw extracted strings and normalizes them. Pack size `"4X5LB"` becomes `{ quantity: 20, unit: "lb" }`. Price `"$14.99"` becomes `14.99`. Category gets validated against a known enum.

Agent C calls no API. It's regex, type coercion, and lookup tables — deterministic code that runs in microseconds.

## Why that's actually the right design

The temptation when building "an AI pipeline" is to make every stage agentic. There's a kind of architectural symmetry that makes it feel right — three problems, three LLM calls.

But Agent C's job isn't understanding. It's transformation. The ambiguity is already gone by the time data reaches it — Agent B figured out that column 4 is price. Agent C just needs to strip the dollar sign and parse a float. There's no judgement required, and therefore no reason to pay for judgement.

This same logic extends to catalog matching. When a bid comes in and we need to find which of our ~270 inventory items best matches "BNLS SKNLS CHKN BRST 6OZ," we have two tools available:

- **Vector embeddings:** Pre-computed once, stored in the database. Matching is a SQL cosine similarity query — sub-millisecond, no API call, scales infinitely.
- **LLM normalization:** Send the abbreviated name to the model, get back "Chicken Breast Boneless Skinless 6oz," try matching again.

The right answer is embeddings first, LLM only when similarity is ambiguous. Embeddings handle the semantic gap between abbreviation and full name without any inference cost. The LLM handles the edge cases the embedding space can't bridge, and the results get cached so the same abbrevation never costs twice.

## The pattern

Each stage in a processing pipeline has a different cognitive requirement:

| Stage | Requirement | Right tool |
|---|---|---|
| Document layout analysis | Visual reasoning, holistic view | Vision model |
| Sequential data extraction | Context continuity across pages | LLM with message history |
| Field normalization | Deterministic transformation | Regex + type coercion |
| Catalog matching (semantic) | Similarity in meaning space | Pre-computed embeddings |
| Name disambiguation (edge cases) | Language understanding | LLM call, cached |

"AI pipeline" doesn't mean every node calls a model. The architecture should fit the cognitive shape of each step.

When we started writing this down, we assumed we'd be explaining why each agent uses AI. Instead we ended up explaining why the third one doesn't — and that explanation turned out to be cleaner, cheaper, and easier to test than the alternative.

The box labeled "Agent C" in our diagram has no arrow pointing to an API endpoint. That's not a gap we forgot to fill. It's the design.

