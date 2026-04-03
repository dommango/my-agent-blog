---
layout: post
title: "Five Ways Automated Agents Fail Silently (And How to Fix Each One)"
date: 2026-04-03 15:16:08 +0000
categories: [debugging]
tags: [agents, prompt-engineering, debugging, automation]
excerpt: "A catalog of five concrete failure modes that let agent prompts run to completion without doing any actual work — with targeted fixes and reusable patterns for each."
---

We ran a scheduled agent that checks a changelog, updates an HTML file, and opens a pull request. It ran every morning for a week. It logged success every time. It missed two full releases.

When we dug in, the agent hadn't crashed — it had *reasoned its way* past the work it was supposed to do. That's the thing about agent failures: they rarely throw exceptions. They just quietly decide there's nothing to do.

Here are the five specific failure modes we found, and what we did about each one.

---

## 1. The Version Tag It Wrote Itself

**What happened:** The agent tagged newly added items with `<!-- added:v2.1.88 -->` even when those items came from the v2.1.89 changelog. On the next run, it read the HTML, saw v2.1.89 content already present, and concluded the placemat was current.

**Why it's subtle:** The agent wasn't wrong about the facts — the content *was* there. It just labeled it with the wrong version. The data it relied on to make the "nothing to do" decision was data it had created incorrectly on a previous run.

**The fix:** Make the version tag instruction unambiguous. Instead of "include the tracking comment `<!-- added:vX.Y.Z -->`," write: "The version in that comment must be the CC changelog version where the feature *first appeared*, not the current placemat version. If the placemat is at v2.1.88 and you're adding a v2.1.89 feature, write `<!-- added:v2.1.89 -->`."

**Reusable pattern:** Never let an agent use its own prior outputs as the source of truth for whether work is needed. Use a canonical external source — the release tag was only trustworthy before the agent started writing it.

---

## 2. Naive Version Comparison

**What happened:** Step 3 of the prompt said: "If no new versions exist, output 'Placemat is current' and stop." The agent compared the release tag span against the latest changelog entry. They matched (both said v2.1.88), so it stopped — even though the actual v2.1.89 and v2.1.90 content was missing.

**Why it's subtle:** The comparison logic was correct in isolation. The bug was that the *data being compared* had drifted from reality.

**The fix:** Add a cross-check step. For each changelog bullet you'd classify as relevant, grep for its key term in the HTML. If relevant items from a newer version are already present, skip them — but do *not* treat their presence as evidence the placemat is fully current. The release tag is only authoritative when combined with a content audit.

**Reusable pattern:** Checkpoint comparisons against multiple independent signals. "Version tag matches" + "expected content present" + "no known missing items" is a much stronger stop condition than any single check alone.

---

## 3. The Self-Review Loop With No Exit

**What happened:** Step 7 of the prompt said: "If ANY check fails, fix the issue and run through the full checklist again. Do not proceed until all checks pass." No iteration limit. The agent likely hit a structural issue in the HTML, re-reviewed, hit the same issue, re-reviewed — and never reached the commit step.

**Why it's subtle:** The intent was good: ensure quality before shipping. But "retry until passing" without a cap is a divergent loop. The agent ran for its full context window and exited cleanly without producing any output.

**The fix:** Add a hard cap. "Maximum 3 review iterations. If checks still fail after the third pass, note the remaining issues in the PR description and proceed anyway."

**Reusable pattern:** Any retry or re-check loop in a prompt needs an exit condition beyond success. Agents don't get tired — they'll retry indefinitely. Build in a "best effort" path that ships partial work with a visible failure note.

---

## 4. Branch Collision From a Previous Failed Run

**What happened:** The previous run had pushed a branch — `claude/update-v2.1.90` — but failed before creating the PR. When the next run tried to push to the same branch name, git rejected it with a non-fast-forward error. The agent hit the error, had no recovery logic, and exited.

**Why it's subtle:** The failure message was clear in the logs, but there were no logs visible — the push failure just meant no PR appeared. From the outside, the run looked clean.

**The fix:** Before pushing, check whether the remote branch exists and delete it if so:

```bash
git ls-remote --exit-code origin claude/update-v2.1.90 && \
  git push origin --delete claude/update-v2.1.90 || true
git push -u origin claude/update-v2.1.90
```

**Reusable pattern:** Any agent that creates named resources (branches, files, database rows) needs to handle the case where that name already exists. Treat idempotency as a first-class concern in the prompt.

---

## 5. Fetch Failure That Looked Like Success

**What happened:** On one run, the changelog URL returned a truncated response — no version headers, just partial HTML. The agent found no new versions (because the version headers weren't there), output "Placemat is current," and stopped.

**Why it's subtle:** The agent followed its instructions correctly. The instructions just hadn't anticipated empty input.

**The fix:** Add an explicit validation step after the fetch:

> "If the fetched content contains no version headers (lines matching `## v2.X.YY`), assume the fetch failed or was truncated. Retry once. If it still fails, open a PR titled 'Agent: changelog fetch failed on [date]' with the error details."

**Reusable pattern:** Validate that fetched data is structurally plausible before acting on it. A successful HTTP 200 doesn't mean the content is usable. Fail loudly — a PR with a failure note is infinitely more useful than a silent no-op.

---

## The Common Thread

All five failures share the same shape: the agent reached a legitimate-looking decision point, made a locally reasonable choice, and stopped — without ever producing the intended output. No crash, no error, no alert.

The defense isn't better error handling in the traditional sense. It's designing prompts so that *non-action has to be explicitly justified*, not just the default when something looks ambiguous. Add caps to loops. Validate inputs. Check for branch collisions. Make failure visible. 

An agent that opens a PR saying "I failed here's why" is worth ten agents that quietly succeed at nothing.

