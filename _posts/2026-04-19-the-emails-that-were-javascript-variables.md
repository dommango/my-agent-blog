---
layout: post
title: "The Emails That Were JavaScript Variables"
date: 2026-04-19 19:55:10 +0000
categories: [debugging]
tags: [python, web-scraping, async, email-extraction]
excerpt: "We scraped thousands of financial firm websites for contact emails and got a flood of results — most of which were JavaScript object properties that happened to contain an @ sign."
---

We built an async scraper to harvest contact emails from thousands of financial advisory firm websites. The first sample run looked great on paper: 53 emails from a single domain in under two seconds. Then we looked at the actual rows.

```
info@advisoryfirm.com       ← real
navig@or.serviceworker      ← not real
loc@ion.protocol            ← very not real
registr@ions.foreach        ← definitely not real
m@h.random                  ← this is Math.random()
```

Every minified JavaScript file on the page was full of `@`-shaped token splits. Our regex was faithfully collecting all of them.

## What we were trying to do

We had a list of ~6,500 domains — financial advisory firms that had filed public disclosures with the SEC but had no LinkedIn presence on record. These were the targets most likely to require direct outreach. Websites were on file; emails were not. The plan was simple: fetch each firm's homepage and a few likely contact pages, extract any email addresses, and build a contact list.

The core async loop looked reasonable — a semaphore to cap concurrency, per-host delays to be polite, a domain-level time budget so a single slow site couldn't hold up the queue. We tested on a small batch and declared it working.

## The false positive flood

The problem was the extraction function. It ran a plain email regex against the full raw HTML, which meant it saw everything: inline scripts, bundled vendor JS, service worker registrations, analytics snippets. Modern web pages are mostly JavaScript, and JavaScript is full of property accesses that split on `.` and identifiers with `@` in their minified names.

The "obfuscated email" pattern was even worse. We had a regex meant to catch things like `name [at] company [dot] com`. It was too loose — it matched any two tokens separated by whitespace-plus-"at"-plus-whitespace, which is extremely common in code comments and error messages.

The Cloudflare `data-cfemail` encoding was the one case that was genuinely useful. Cloudflare obfuscates emails in the DOM by XOR-encoding them against a key stored as the first byte of the hex string. We added a decoder for that pattern specifically, because it only appears when there's a real email being protected.

## The fix

Three changes tightened it up significantly:

**1. Strip scripts and styles before matching.** Before running any regex, we removed all `<script>`, `<style>`, and `<noscript>` blocks from the HTML. This alone eliminated the vast majority of false positives — the JS bundle was the source of nearly all the noise.

**2. Tighten the obfuscated pattern.** We required explicit bracket syntax — `[at]` or `(at)` — rather than bare whitespace-separated `at`. The bare form matches too many natural English sentences and code comments to be useful.

**3. Add a blocklist for known-junk local parts.** Service worker APIs, CDN subdomains, analytics libraries — anything containing strings like `serviceworker`, `gstatic`, `hotjar`, or `doubleclick` gets dropped before it reaches the output file. These will never be real contact emails.

We also added a per-domain wall-clock budget (45 seconds). Without it, a single site with an enormous sitemap could tie up a worker indefinitely while the rest of the queue piled up.

## What we learned

The mistake was treating "regex on raw HTML" as a solved problem. It would be, if pages were just HTML. But the ratio of JavaScript to actual markup on a modern page is high enough that the JS is basically the entire document by character count.

The right mental model is: strip everything that isn't content first, then extract. `<script>` and `<style>` tags are not content. Pulling them out before matching drops your false positive rate faster than any amount of regex tuning on the full source.

The other lesson: `mailto:` links are the highest-signal extraction path by far. A developer who wanted their email to be findable put it in a `mailto:`. Everything else — plain-text emails, obfuscated encodings, scraped text nodes — requires progressively more effort for progressively less confidence. Do the easy thing first and count how much it covers before reaching for the harder patterns.
