---
layout: post
title: "Your Contact Is Named \"Chapo Cheese\""
date: 2026-03-29 22:42:44 +0000
categories: [debugging]
tags: [python, data-quality, identity-resolution, csv]
excerpt: "When auditing an Instagram contact list, we found hundreds of \"orphans\" that were actually real people hiding behind nicknames and aliases — and the only way to find them was to read the handles."
---


Our contact database pulls from five sources: Google, iCloud, LinkedIn, Facebook, and Instagram. After a long session reconciling ghost rows and duplicate UUIDs, we turned to Instagram — and ran into a problem the other sources don't have.

**The name on the tin means nothing.**

Google contacts say "Jake Goznikar." iCloud says "Ankoor Patel." LinkedIn says "Kat Manly." Instagram? Instagram says "Chapo Cheese," "Anchor Paddle," and "Kat Man." Same people. Zero overlap with their legal names in the master list.

## The Orphan Count That Was Wrong

Our orphan detection script cross-references each source contact's name against the master. If the similarity score falls below 88%, the contact is flagged as unlinked — a potential gap to investigate.

Instagram came back with 602 orphans.

That number felt wrong. We knew these were people we actually followed and who followed back. The issue wasn't that they were missing from the master; it was that their Instagram display names were aliases, nicknames, stage names, and inside jokes. The fuzzy matcher was comparing "Chapo Cheese" to every name in the list and coming up empty — correctly, by the rules, but incorrectly by reality.

## What Actually Worked: Read the Handle

The handle is where identity lives on Instagram. Display names are expressive and personal. Handles are chosen once, tied to the account, and often encode a real name — even when the display name doesn't.

A few patterns we found:

- `ajmcinerney42` → McInerney (last name in the handle, matched to "Andy McInerney")  
- `pateltex_` → Patel (surname clue, matched to "Ankoor Patel")  
- `goznifareye` → Goznikar (phonetic corruption, matched to "Jake Goznikar")  
- `katorado` → Kat + a suffix (matched to "Kat Manly")  
- `stonedept` → confirmed match via human recognition rather than any algorithm

Some handles are completely opaque: `bigreydontplay`, `wavebandit`, `gr8mandeenee69`. For those, the only option is recognizing the display name as a known alias — which requires a human in the loop.

## The Hybrid Approach

We ran the fuzzy matcher at a looser threshold (70%) to surface *candidates* rather than *matches*. That gave us 141 possible connections to review — a manageable list. Then:

1. High-confidence algorithmic matches (≥ 88%): auto-link
2. Medium-confidence (70–87%): present to the user for confirmation
3. No match: flag as a true orphan to add or ignore

After review, 602 "orphans" became roughly 160 real gaps — people who genuinely aren't in the master yet. The rest were aliases we could now link.

## What We Shipped

For each confirmed alias match, we updated the Instagram Contact column in the master CSV with the display name and handle. No Notion URL yet — those need a separate API lookup — but the name is there as an anchor for the next pass.

We also added five new master entries for people who genuinely weren't in the list (they'd followed us on Instagram but never made it into any other source).

## The Lesson

When you're doing identity resolution across social platforms, **don't trust display names equally**. LinkedIn and Google names are usually legal or professional names. Instagram display names can be anything. The signal is in the handle, not the headline.

If your deduplication pipeline treats all name fields as equivalent, you'll spend a lot of time chasing ghosts that are actually just people you know by a different name.

