---
layout: post
title: "The Scraper That Thought HTTP Was a Hostname"
date: 2026-04-20 00:10:40 +0000
categories: [debugging]
tags: [python, web-scraping, data-quality, asyncio]
excerpt: "A case-sensitivity bug in URL normalization caused our scraper to embed the URL scheme inside the hostname — and it took a quick connectivity smoke test to find it."
---

We wanted emails. Six-and-a-half thousand financial advisory firms, each with a website listed in their SEC regulatory filing, and not a single email address in the government data — the SEC doesn't include them in the public bulk export. So we built a scraper.

The plan was straightforward: pull the candidate domains from the Form ADV bulk filings, spin up an async Python crawler with `httpx`, hit a handful of likely pages per domain (`/`, `/contact`, `/team`, `/about`), and regex out the `mailto:` links. Thirty concurrent domains, 45-second budget per site, resumable state so a crash doesn't restart from zero. We estimated about two hours for the full run.

The first sample — 20 domains — returned one email.

## Something Was Very Wrong

Nineteen of the twenty domains came back as `conntimeout` or `connerr`. We checked the scrape log and spotted a site we knew was live: `harolddance.com`. We'd hit it manually just minutes before and it loaded in under a second. But our scraper was timing out on it repeatedly.

We added a quick smoke test — eight domains, raw `httpx` call, time it:

```python
async with httpx.AsyncClient() as c:
    r = await c.get(url, timeout=8.0, follow_redirects=True)
    print(f"{url}  {r.status_code}")
```

`harolddance.com` responded in 0.6 seconds. Fine. So what URL was our scraper actually fetching?

We logged the normalized URL before the request went out. It read:

```
https://HTTP://WWW.HAROLDDANCE.COM
```

There it was. The SEC bulk data stores website URLs exactly as firms typed them into their Form ADV filings over the years. This particular firm — like a few hundred others — had typed `HTTP://WWW.HAROLDDANCE.COM` in all caps. Our normalization function checked whether the URL started with `http://` or `https://` using a case-sensitive comparison. It didn't match, so we prepended `https://`. Now we had a URL where the scheme portion of the original had become part of the hostname. `httpx` was dutifully trying to open a TCP connection to a host called `HTTP`.

## The Fix Was Eight Lines

```python
def normalize_url(base: str) -> str:
    base = base.strip()
    if not base:
        return ''
    low = base.lower()
    if low.startswith('http://'):
        base = 'http://' + base[7:]
    elif low.startswith('https://'):
        base = 'https://' + base[8:]
    else:
        base = 'https://' + base
    # Lowercase the host portion too
    p = urlparse(base)
    host = (p.netloc or '').lower()
    return f'{p.scheme}://{host}{p.path}'.rstrip('/')
```

Check the scheme case-insensitively. Lowercase the host. Strip. Done.

After the fix, the same 20-domain sample went from 1 hit to 12 — a 60% success rate. We ran the full 6,505-domain list overnight. Two hours and five minutes later: **10,291 emails from 4,126 firms**, at a 63.5% hit rate across the dataset.

The emails came from four sources in priority order: explicit `mailto:` links (76%), plain regex on visible page text (18%), Cloudflare's `data-cfemail` encoding which we decoded manually (6%), and the occasional "name [at] domain [dot] com" written out in prose. We'd also learned earlier in the session that minified JavaScript contains token sequences like `navigator.serviceworker.getregistrations` that look suspiciously like email addresses — stripping `<script>` tags before running any regex cut the false-positive rate dramatically.

## The Lesson

When your upstream data source is fifteen months of government CSV files typed by humans at thousands of small businesses, assume the inputs are broken until proven otherwise. Uppercase schemes, missing schemes, bare domains, LinkedIn URLs passed off as company websites — we hit all of them. The normalization layer isn't plumbing. It's the first line of real engineering in the pipeline.

The good news: once you've handled the weird cases, the data is remarkably consistent. SEC Form ADV is a standardized filing. Ninety percent of the URLs were fine. It was the ten percent that sent our scraper into a wall of timeouts — and they were all variations of the same mistake: someone at a firm, years ago, typed their URL in caps.
