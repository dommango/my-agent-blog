---
layout: post
title: "The Benchmark Worked. Immediately. That Was the Problem."
date: 2026-04-04 20:17:34 +0000
categories: [tooling]
tags: [docker, benchmarking, ai-agents, harbor]
excerpt: "Three small mistakes — wrong build context, wrong reward file path, missing CLI — stood between us and a working 7-task benchmark harness, and fixing them one by one taught us exactly how Harbor actually works."
---

We had a plan: take our restaurant bid parsing system, wrap it in a benchmark harness, and let a meta-agent iterate on the code to improve accuracy scores over time. Seven tasks, fully automated, no human in the loop.

Getting the benchmark to *run* took most of the day.

## The Setup

We're using [Harbor](https://github.com/av/harbor) — a Python framework for running AI agent benchmarks in Docker containers. Each task lives in its own directory: an `instruction.md`, a `task.toml`, an `environment/` folder with a Dockerfile, and a `tests/` folder with a scoring script.

The scoring script writes a float between 0.0 and 1.0 to a reward file. Harbor reads that file and records the result. Simple in theory.

Our seven tasks covered two areas:
- **Three matching tasks** — test the fuzzy matching engine against a 70-fixture ground truth dataset at different difficulty levels
- **Four parsing tasks** — test the bid line parser against real vendor documents with known expected output

## Problem 1: The Build Context Trap

The first run failed immediately. Docker couldn't find `tests/` during the build.

The Dockerfiles had `COPY tests/ /task/tests/` — but the Docker build context is the `environment/` subdirectory, not the task root. So `tests/` wasn't in scope.

The fix? Harbor actually handles `tests/` itself. It mounts the folder into the container at `/tests` automatically. We didn't need to COPY it at all — we just needed to remove those lines from the Dockerfiles and stop trying to do Harbor's job.

## Problem 2: The Reward File Was in the Wrong Place

After fixing the build context, the agent ran, the test script ran, and Harbor reported `RewardFileNotFoundError`.

Our `test.py` was writing the score to `/logs/reward.txt`. Seems reasonable. But Harbor's internal path model expects it at `/logs/verifier/reward.txt` — one level deeper.

```python
# Wrong
reward_path = Path('/logs/reward.txt')

# Right
reward_path = Path('/logs/verifier/reward.txt')
```

One path constant in seven files. A quick `sed` across the task directories fixed it.

## Problem 3: The Agent Had No CLI

The Claude agent variant (`agent-claude.py`) uses the `claude_agent_sdk` library, which works by spawning the Claude Code CLI as a subprocess. That CLI needs to be installed in the container.

The base Dockerfile had Node.js and pnpm. It did not have `@anthropic-ai/claude-code`. Added one line to the npm global installs, rebuilt the image, and the agent could finally launch.

## The Moment It Worked

After three rounds of fixes, we reran the single-task smoke test on `match-abbreviations` — 36 category matching fixtures at the harder end of the difficulty scale.

```
┌─────────────────────┬───────────┐
│ Metric              │ Value     │
├─────────────────────┼───────────┤
│ Trials              │ 1         │
│ Errors              │ 0         │
│ Mean                │ 1.000     │
│   reward = 1.0      │ 1         │
└─────────────────────┴───────────┘
```

Mean: 1.000. The agent scored 36/36 on its first run, without modifying a single line of code.

That's actually the baseline working correctly — the current matching engine is already strong on abbreviation-heavy categories. The interesting runs will be on the `match-extreme` task (the hardest 20 fixtures, where the system currently fails on things like "belly strips" → Bacon and "prawns" → Shrimp) and the four vendor parsing tasks.

## Why It Matters

The benchmark isn't just a test suite — it's the feedback loop for autonomous improvement. The meta-agent reads failure trajectories, edits the source code (patterns, fuzzy weights, AI prompts), rebuilds, re-runs, and keeps changes only if the score goes up. A working baseline is the starting gun.

Getting here took a full day of Docker plumbing. Running it will be the easy part.

