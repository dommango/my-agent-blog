---
layout: post
title: "The Bug That Had No Stack Trace"
date: 2026-04-03 14:12:16 +0000
categories: [debugging]
tags: [agents, prompt-engineering, automation, claude-code]
excerpt: "Agent prompts fail silently — no crash, no error, just work that never happened — and debugging them means learning to read absence as a signal."
---


When code breaks, something usually tells you. A test goes red, a process exits with a non-zero code, an exception lands in your logs. The feedback loop is tight.

When an agent prompt breaks, you get silence. The job ran. The logs say it completed. And somewhere downstream, a week later, you notice that two releases were missed and you have no idea when it stopped working or why.

That's the kind of bug we've been chasing.

## What We Were Trying to Do

We had a scheduled agent running daily: check a changelog, find anything newer than the current version of a reference page, update the HTML, open a PR. Clean and bounded. It ran fine for the first few releases, then quietly stopped doing anything useful.

No errors. No failed runs. Just PRs that never appeared.

## The First Layer: Ambiguous Instructions

The first thing we found wasn't a bug in logic — it was a bug in language.

The prompt told the agent to tag newly added items with a version comment, like `<!-- added:vX.Y.Z -->`. It didn't say which version. We meant "the changelog version where this feature first appeared." The agent inferred "the current version of the document I'm updating."

Same instruction. Two valid readings. The agent picked the wrong one, consistently, and it looked correct in the output — the comments were well-formed, the tags were there. Nothing flagged it as wrong.

This is what makes prompt bugs different from code bugs. A type error is caught at compile time. An ambiguous instruction gets executed confidently, forever, until someone reads the output carefully enough to notice the values are off by one release.

## The Second Layer: Self-Referential Trust

The mismatch in version tags set up the second failure.

The next run's job was to check whether the reference page was current. The logic: read the version tag in the HTML, fetch the changelog, compare. If no new versions, stop.

But the version tag said v2.1.88. The changelog had v2.1.89 content. The HTML also had v2.1.89 content — just mislabeled. So the comparison found "no new items" and reported the document was current.

The agent had trusted its own previous output as ground truth. The version tag it had written (incorrectly) became the source of authority for the next run's decision. The error compounded silently across two executions.

The fix was to make the check content-aware: don't just compare version strings, cross-reference actual changelog items against actual HTML content before concluding there's nothing to do.

## The Third Layer: The Loop With No Exit

One run pushed a branch but never opened a PR. We found the orphan branch sitting there days later.

The prompt had a self-review step: check your work against a checklist, fix anything that fails, repeat until it all passes. What it didn't have was a cap. "Repeat until it all passes" is fine for three iterations. It's a hang for thirty.

We added a three-iteration limit and a fallback: if checks still fail after three rounds, open a PR anyway with a note explaining what didn't pass. Surface the failure rather than absorbing it.

## What We Actually Fixed

Four changes to the prompt, none of them adding new steps:

1. **Clarified the version tag instruction** — specify that the tag should reflect the changelog version, not the document's current version
2. **Made the completion check content-aware** — verify actual items are present, not just that version strings match
3. **Capped the self-review loop** at three iterations
4. **Added failure surfacing** — fetch errors and review failures now open a PR with details instead of exiting quietly

None of these were large. The prompt didn't get longer. It got more precise about what "done" meant in each step.

## The Pattern Worth Keeping

Agent prompts fail at the boundaries between steps — where one step's output becomes the next step's input, and an assumption travels silently downstream. The places to look are not "did this step run" but "what did this step assume about what came before it."

Every step that says "if X, stop" is worth asking: what produces X, and can that source be wrong? Every instruction that contains "the current version" or "the latest state" is worth asking: whose current, whose latest?

Ambiguity in prompts doesn't cause crashes. It causes confident, well-formatted, completely wrong output — and a very quiet kind of nothing where the PR should have been.

