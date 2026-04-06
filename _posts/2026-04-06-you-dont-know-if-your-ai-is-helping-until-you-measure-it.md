---
layout: post
title: "You Don't Know If Your AI Is Helping Until You Measure It"
date: 2026-04-06 13:40:39 +0000
categories: [architecture]
tags: [evals, llm, typescript, testing]
excerpt: "Adding Haiku to a parsing pipeline felt like progress — until we realized we had no way to tell whether the AI path was actually better than the one it replaced."
---

## What Happened

We'd spent several sessions adding AI to our bid sheet parsing pipeline — Haiku for vendor detection, name cleanup, column validation, anomaly detection. The code was clean, the tests passed, the confidence scores looked better on our demo data. We were feeling good.

Then someone asked the obvious question: *Is the AI path actually more accurate than what we had before?*

We didn't know. Not precisely. We had intuitions, a handful of manual spot-checks, and some before/after eyeballing. But we had no number. No reproducible comparison. No way to tell if a future change made things better or worse without doing it all again by hand.

That's when we realized we needed an eval framework — not as a nice-to-have, but as the thing that makes every other AI decision meaningful.

## The Problem With "Looks Better"

When you add an LLM to a pipeline, the failure modes are subtle. The model doesn't crash. It doesn't throw an exception. It returns something plausible-looking that's quietly wrong in ways that only show up at scale or in edge cases.

Our pipeline had three processing paths by this point:

- **Quick Parse** — regex + fuzzy matching, fast, deterministic
- **Agent Parse** — three-agent vision pipeline for scanned PDFs and photos
- **Spreadsheet Import** — column detection + AI remap + AI name cleanup

Each path had been tuned independently. Each felt like an improvement at the time it was built. But without a shared benchmark, we couldn't answer: *Which path should we route this document through? When does Agent Parse justify its 10-second latency? How much does AI column remap actually help vs. the baseline detector?*

The answer to all of those questions is the same: you need labeled test cases and a runner that scores them.

## What We Built

The eval framework has three pieces.

**A workflow registry** — a catalog of named workflows (quick-parse, agent-parse, spreadsheet-import) with their input types, expected output shapes, and any configuration they need. Think of it as the menu of things you can benchmark.

**A workflow runner** — takes a workflow name, a set of labeled test cases, and produces scored results. Each test case has a raw input (bid text, spreadsheet rows, an image path) and a ground-truth output (the correctly parsed items). The runner calls the workflow, diffs the output against ground truth, and computes field-level accuracy.

```typescript
// Simplified — the real version handles partial credit and confidence
async function runEval(workflow: string, cases: EvalCase[]): Promise<EvalResult> {
  const results = await Promise.all(
    cases.map(async (c) => {
      const output = await runWorkflow(workflow, c.input)
      return scoreOutput(output, c.expected)
    })
  )
  return summarize(results)
}
```

**An A/B comparison script** — a CLI that runs two workflows against the same test set and prints a side-by-side accuracy table. When we're deciding whether to enable AI column remap by default, we run this. When we tune a prompt, we run this. It takes about 30 seconds and gives us an actual number.

## What Surprised Us

The test case collection turned out to be the hard part. Writing the runner was straightforward TypeScript. But curating labeled examples — real vendor bid sheets with verified correct outputs — took time and forced decisions we'd been deferring.

What counts as a correct match when the vendor calls it "BNLS SKNLS CHKN BRST 6OZ" and our catalog has "Chicken Breast, Boneless Skinless"? Is a 90% confidence score a pass or a fail? Does a misassigned pack size matter as much as a misidentified product name?

Answering those questions made our confidence scoring more honest. We'd been tuning it to feel right. The eval forced us to define "right" precisely enough to automate.

## Why It Matters

An eval framework does something subtle beyond measuring accuracy: it makes the AI integration *falsifiable*. You can now say "this prompt change improved product name extraction by 4 percentage points on our test set" and mean it. You can merge a change and know, not hope, that it didn't regress anything.

Without that, every AI improvement is a guess dressed up as intuition. The pipeline gets harder to reason about with every addition, and the humans maintaining it slowly lose confidence in what they've built.

Measurement isn't the opposite of moving fast. It's what lets you keep moving fast after the easy wins are gone.

