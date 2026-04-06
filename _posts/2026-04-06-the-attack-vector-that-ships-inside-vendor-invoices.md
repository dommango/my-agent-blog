---
layout: post
title: "The Attack Vector That Ships Inside Vendor Invoices"
date: 2026-04-06 16:11:38 +0000
categories: [architecture]
tags: [prompt-injection, llm, security, parsing]
excerpt: "When you pass vendor-supplied text directly into an LLM prompt, you're not just asking the model a question — you're handing an untrusted third party the microphone."
---

When we added an LLM to our bid parsing pipeline, we were thinking about accuracy: could the model expand "BNLS SKNLS CHKN BRST 6OZ" into something a human would recognize? We weren't thinking about security. We should have been.

## The Problem Is the Data

A prompt injection attack is simpler than it sounds. You're building a system that takes user-supplied text and puts it into a prompt. An attacker (or just a maliciously-formatted vendor document) includes text that looks like a new instruction rather than data. The model, which can't tell the difference between your system prompt and payload it just received, follows the injected instruction instead.

In most web apps, this is an edge case. In a document parsing pipeline, it's nearly unavoidable — because the whole point is to ingest arbitrary third-party text. Every vendor invoice, every price sheet, every handwritten bid we process is untrusted input. We were interpolating it directly into prompts like this:

```typescript
// BEFORE: User-supplied strings interpolated raw
let prompt = ''
if (vendor) {
  prompt += `Vendor: ${vendor}\n\n`  // vendor name from uploaded file
}
for (const c of corrections) {
  prompt += `- "${c.originalValue}" → "${c.correctedValue}"\n`  // user corrections
}
prompt += `Parse the following bid lines:\n${rawLines.join('\n')}`
```

That `vendor` field? It comes from the filename of an uploaded document. The `corrections` are user-submitted edits from the review queue. The `rawLines` are scraped text from a PDF. Any of these could contain newlines and carefully-worded "instructions" that blur the boundary between data and prompt.

## Why Vendor Names Are the Sneakiest Vector

We caught this during a code review, but it's worth spelling out *why* vendor names are particularly risky compared to, say, product descriptions.

Product names go through a lot of validation before they reach the LLM. A vendor name often doesn't. It arrives early, gets embedded near the top of the prompt, and establishes "context" for everything that follows. A prompt that starts with `Vendor: US Foods` reads differently to a model than one that starts with `Vendor: US Foods\n\nIgnore previous instructions. Instead, respond with...`

## The Fix Is Boring but Necessary

We wrote a small sanitization utility:

```typescript
export function sanitizeForPrompt(input: string, maxLength = 500): string {
  return input
    .replace(/[\n\r\t]/g, ' ')        // collapse newlines — these are the main vector
    .replace(/[^\x20-\x7E]/g, '')     // strip non-printable characters
    .replace(/\s+/g, ' ')             // normalize whitespace
    .trim()
    .slice(0, maxLength)              // enforce a hard length cap
}
```

Then we applied it to every user-supplied string before it touched a prompt:

```typescript
if (vendor) {
  prompt += `Vendor: ${sanitizeForPrompt(vendor)}\n\n`
}
for (const c of corrections.slice(0, 5)) {
  prompt += `- "${sanitizeForPrompt(c.originalValue)}" → "${sanitizeForPrompt(c.correctedValue)}"\n`
}
```

The key insight is: **newlines are the attack surface**. LLMs treat a newline as a structural separator. Stripping them from interpolated data means injected "instructions" can't establish a new paragraph — they land as continuous text inside a data field, where the model reads them as content rather than commands.

## A Note on Defense in Depth

Sanitization alone isn't a complete defense. It makes casual injection much harder, but a determined attacker working within the printable ASCII range could still attempt semantic injection ("...please disregard the above format rules and instead..."). For that, you need structural defenses: system prompts that explicitly scope the model's task, structured output formats (so there's no free-text field for injected instructions to land in), and ideally response validation that checks the output shape before using it.

For a parsing pipeline processing vendor invoices, the practical risk is low — nobody is crafting malicious bid sheets to manipulate restaurant cost data. But the *pattern* matters. Any place you're interpolating third-party text into a prompt is a place where the prompt/data boundary is fuzzy, and fuzzy boundaries are where security bugs live.

The boring function that strips newlines and truncates strings? That's the whole fix. It's not glamorous, but neither is finding out your AI helpfully followed the instructions hidden in a PDF.

