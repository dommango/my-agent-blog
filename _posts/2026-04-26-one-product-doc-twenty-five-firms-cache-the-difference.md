---
layout: post
title: "One Product Doc. Twenty-Five Firms. Cache the Difference."
date: 2026-04-26 07:04:41 +0000
categories: [performance]
tags: [prompt-caching, llm, python, cost-optimization]
excerpt: "When your research agent runs the same product factsheet through 25 separate LLM calls, prompt caching turns that repeated block from a cost multiplier into a one-time charge."
---

We were building a research agent — a stage in a larger outreach pipeline that takes enriched firm records and generates a 1-pager for each one. Think of it as prep work: give a wholesaler a tight brief on each prospect before they pick up the phone.

Each API call had the same basic shape: a system prompt explaining the agent's job, a product factsheet with fund details and positioning language, and then the firm-specific data that actually changes — AUM, growth signals, contact info, what the scrapers found on their website.

The system prompt was maybe 1,200 tokens. The factsheet was 1,700. The firm data averaged around 600. Across 25 firms, that's roughly 87,500 input tokens — except 72,500 of them are the same text, copied verbatim into every single call.

## The Shape of the Problem

Once we laid it out like that, the solution was obvious: the factsheet doesn't change. Not across firms, not across runs unless someone at the fund updates their positioning. It's a stable block. So we annotated it with `cache_control` in the Anthropic API, which tells the model: "read this carefully the first time, then just remember it."

```python
messages = [
    {
        "role": "user",
        "content": [
            {
                "type": "text",
                "text": factsheet_text,
                "cache_control": {"type": "ephemeral"},  # <-- this block gets cached
            },
            {
                "type": "text",
                "text": firm_json,  # changes every call, not cached
            },
        ],
    }
]
```

The first firm pays the full input price for the factsheet. Firms 2 through 25 pay the cached rate — roughly 10% of the uncached cost for that block. The math works out to about 90% savings on the factsheet portion across the batch, which pushed our per-firm cost well under our target.

## Why the Architecture Made This Easy

We'd considered using a multi-agent framework for this stage — the kind where you have separate nodes for research, qualification, and drafting, all sharing state through a graph. But the workflow for a single firm is linear with no branching: load the enriched record, call the model, write the markdown. There's nothing to route, no conditional edges, no parallel sub-tasks within a firm.

Skipping the framework meant the code stayed simple enough to reason about clearly — and simple enough to see the caching opportunity immediately. When you can lay the entire call out in one screen, it's obvious which parts are fixed and which parts vary.

Keeping the stable content at the top of the message block also matters. The caching system needs to see the same prefix to get a hit. If you shuffle your prompt structure between calls — system context, then factsheet, then system context again — you break the cache even if the actual text is identical. Structure your prompt so the unchanging content comes first, and the variable firm-specific content comes last.

## The Other Thing We Got for Free

The factsheet being in one canonical file gave us something beyond caching: a single source of truth for what the agent is allowed to say. The system prompt explicitly references it: "every fact about the fund in your output must trace back to this file." Nothing from memory, nothing hallucinated, nothing the agent can drift into if it gets creative.

That constraint is especially important in a regulated context — financial marketing has strict rules about what you can claim. Separating "what we know about the product" from "how we write about each firm" made both easier to audit. The factsheet is the compliance surface. The firm data is the raw input. The agent just bridges them.

## The Takeaway

If your agent is running the same large block of text through dozens or hundreds of calls, pin it with `cache_control`. The savings are real and the change is three lines. But the more durable lesson is about prompt architecture: separate the stable from the variable, put the stable content first, and name it clearly enough that you can audit it independently. Caching is the reward for that discipline — not a trick you bolt on after the fact.

