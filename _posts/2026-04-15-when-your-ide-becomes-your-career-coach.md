---
layout: post
title: "When Your IDE Becomes Your Career Coach"
date: 2026-04-15 13:20:11 +0000
categories: [tooling]
tags: [claude-code, cursor, workflow, developer-tools]
excerpt: "We updated a resume using Claude Code and Cursor together — and the seam between the two tools is exactly where the real errors were hiding."
---

We weren't building a feature. We weren't fixing a bug. We were updating a resume.

And yet the workflow looked almost identical to a coding session: an AI assistant reading files, making targeted edits, and handing off to a visual editor for human review.

## The Setup

The career database is a folder of Markdown files with YAML frontmatter — roles, skills, achievements, and a profile summary. No build step, no tests, no deployment pipeline. Just structured text that feeds interview prep, job applications, and performance reviews.

The HTML resume lives alongside it: same content, different format. Both files need to stay in sync when anything changes.

## What Happened

We wanted to update the professional summary. The old version led with job title and years of experience — fine for a recruiter scan, but flat. It buried the actual story: cross-functional execution, regulatory transformation, operating model change at scale.

Claude Code read both files, proposed a revised summary, and made the edit in both places with a single call. Thirty seconds.

Then Cursor opened the HTML file for a final human pass.

That's where it got interesting. Prettier had reformatted the indentation on save — harmless, but visually noisy. And buried in line 242, autocorrect had quietly changed "Citi's platforms" to "cities and platforms." The kind of error that passes every spell check and reads fine at a glance.

## The Lesson About Tool Layering

Each tool in the chain did what it does best:

- **Claude Code** handled the reasoning work — reading context, understanding what the summary was trying to say, rewriting it with the right emphasis, and applying the change consistently across two files.
- **Cursor** handled the visual review — letting a human skim rendered HTML, spot formatting noise, and catch the autocorrect that no automated check would flag.

Neither tool would have caught everything alone. Claude Code doesn't render HTML or notice that "cities" slipped past. Cursor doesn't know that a professional summary across two files needs to stay in sync.

The insight isn't that AI tools are great (they are) or that visual editors are great (they are). It's that **the seam between them is where the real errors live** — and a good workflow makes that seam visible rather than hiding it.

## Why It Matters Beyond Resumes

This pattern shows up anywhere you have structured documents that need to stay consistent: API documentation, onboarding guides, legal templates, config files with human-readable descriptions. The "code" is just text with rules. The tools don't care what it means — but the humans at the review step do.

Build the handoff deliberately. Know what each tool is and isn't good at. And always open the file in a renderer before you ship it.

