---
layout: post
title: "When Your Self-Updating Bot Skips an Update"
date: 2026-04-03 03:52:14 +0000
categories: [debugging]
tags: [automation, agents, debugging, version-management]
excerpt: "A scheduled agent confidently reported \"nothing to do\" and missed two releases — because it trusted the version tag it had written itself, not the actual HTML content it was supposed to be tracking."
---

We had a scheduled agent whose entire job was to keep a reference page up to date. Every morning at 9 AM it would wake up, check the latest changelog, compare it against the current version tag in the HTML, and open a pull request if anything was new. Clean and simple.

Then one day we noticed the page was two releases behind. The agent had run. No PR had appeared. And when we checked the logs, it had signed off with the digital equivalent of a shrug: *"Placemat is current at v2.1.88."*

It was not current at v2.1.88.

## What Actually Happened

A few days earlier, the agent had done its job a little *too* enthusiastically. It pulled in some features from v2.1.89 early — tagging them `<!-- added:v2.1.88 -->` in the HTML — but left the version tag in the header reading `v2.1.88`. The page contained v2.1.89 content, but claimed to be v2.1.88.

So the next time the agent ran, it did exactly what it was told:

1. Read the version tag → `v2.1.88`
2. Fetch the changelog → v2.1.89 and v2.1.90 are newer
3. Scan the HTML for missing v2.1.89 items → already present (because the previous run had snuck them in)
4. Conclude: nothing new for v2.1.89, skip to v2.1.90... and somehow decide the diff was thin enough to not warrant a PR

The agent had corrupted its own source of truth, then trusted that corrupted source of truth to decide there was nothing to do.

## The Sneaky Part

What made this hard to spot was that the agent didn't *fail*. It ran successfully. It checked the right URL, read the right file, compared them intelligently. Every step worked. The output was just wrong, because the premise — "the version tag reflects what's actually in the file" — was no longer true.

This is a class of bug we don't think about enough: **state drift in self-modifying systems**. When an automated process both reads *and* writes the state it uses to make decisions, it can enter a self-consistent-but-wrong equilibrium. The agent's check passed because the agent's previous run had already partially done the work and marked it done.

## The Fix

The real issue was a single source of truth problem. The agent was using the `<span class="release-tag">` in the HTML header as the canonical version — but that span was only as reliable as whoever last updated it.

A more robust approach is to cross-check two independent signals:

- The version tag (what the page *claims* to be)
- A spot-check of specific known items from that release (what the page *actually contains*)

If those two signals disagree, something is off and the agent should flag it rather than quietly deciding the work is done.

We also tightened up the version tagging: the `<!-- added:vX.Y.Z -->` comments in the HTML now get written with the version they were *actually* released in, not the version the agent happened to be processing at the time.

## The Broader Lesson

When you build a bot that updates something and then uses that something to decide what to update next, you've created a feedback loop. Most of the time it hums along fine. But any time the bot writes partial or incorrectly-labelled state, the next run will inherit that mistake and potentially amplify it.

The fix isn't to distrust automation — it's to make your automation's decisions based on sources it didn't write. Read the changelog directly. Compare specific content hashes. Never let a label you placed yesterday be the only thing standing between "update" and "nothing to do."

Your bot might be lying to you, and it doesn't even know it.

