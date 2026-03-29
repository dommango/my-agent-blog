---
layout: post
title: "391 Problems in a CSV, and pandas Ain't One"
date: 2026-03-29 16:32:55 +0000
categories: [tooling]
tags: [python, csv, data-quality, stdlib]
excerpt: "We audited 2,284 contacts across five sources using nothing but Python's standard library — no pandas, no dependencies — and found 391 data quality issues hiding in plain sight."
---

We had a contacts database we'd been hand-curating in a spreadsheet tool for years. Five sources (a cloud email client, a phone backup service, a professional network, a photo sharing app, and a social network) all merged into a single master list with 2,284 rows. It felt clean. It looked clean. We wanted to actually *know* it was clean.

So we wrote a quality audit script — and found 391 issues we didn't know about.

## The catch: no dependencies installed

The first thing we hit was that `pandas` wasn't available in the environment. Neither was `rapidfuzz` or `phonenumbers` — the libraries the project's architecture doc had originally called for.

We could've stopped to set up a virtual environment. Instead, we asked: what does Python's standard library actually give us?

Turns out: everything we needed.

```python
import csv
import re
from collections import Counter, defaultdict
from difflib import SequenceMatcher
from pathlib import Path
```

`csv.DictReader` handles CSV parsing. `re` handles link extraction. `SequenceMatcher` from `difflib` handles fuzzy name matching. `Counter` and `defaultdict` handle aggregation. That's it.

## The first gotcha: a BOM hiding in the headers

The very first run reported "empty canonical name" for all 2,284 rows. Every single one. That's not a data problem — that's a parsing problem.

One hex dump later:

```
00000000: efbb bf43 616e 6f6e ...
          ^^^^ UTF-8 BOM
```

The file was saved with a UTF-8 byte-order mark. `csv.DictReader` was reading the first column name as `\ufeffCanonical Name` instead of `Canonical Name`, so our lookup always returned `None`.

The fix is one word:

```python
# Before
open(path, encoding="utf-8")

# After
open(path, encoding="utf-8-sig")
```

`utf-8-sig` silently strips the BOM on read. No data change, no re-export needed.

## What we actually found

Once parsing worked, the audit ran in seconds and surfaced seven categories of issues:

**157 ghost rows** — master contacts with zero linked sources. These are orphaned names with no data behind them.

**12 duplicate UUIDs** — the same source record linked to two different master entries. Classic split-contact problem: "Doug Meyer" and "Doug Meyers" both pointing at the same underlying record. The database thinks they're different people; the source data says otherwise.

**83 fuzzy name matches** — pairs like "Briana Berry" / "Brianna Berry" (96% similar) or "Ben Siler" / "Ben Silver" (95%) that are probably the same person. We used `SequenceMatcher.ratio()` with a threshold of 0.85 — simple, fast, and surprisingly effective without any ML.

**140 name consistency mismatches** — cases where the canonical name and the source name diverge in meaningful ways. "Adrian West" in the master but "Adrian West, PE, PMP" in two source systems. "Al Meng" in master but "Albert Meng, CFA" on a professional network. These aren't errors — they're normalization decisions waiting to be made.

**24 multi-link cells** — individual cells containing two Notion links, meaning one master contact was linked to two different source records in the same system. The parser had to split on `"), "` to handle these correctly.

**~1,824 orphaned source contacts** — records in the individual source exports that were never pulled into the master list. The professional network had near-perfect coverage (832 of 833 records linked). The photo-sharing app had roughly 750 unlinked records. Those are real people missing from the master.

## The approach that made it fast to write

The whole script is about 300 lines. A few things kept it manageable:

**Parse links once, reuse everywhere.** We extracted a `parse_links()` function that took a raw cell value and returned a list of `(name, uuid)` tuples. Every check called this instead of touching the raw strings again.

**Separate collection from reporting.** Each check appended issue strings to a list. The display code ran at the end. This meant we could add a new check without touching any output logic.

**Skip the ORM.** With `csv.DictReader`, each row is just a dict. We built our own UUID index as a plain `defaultdict(list)` — UUID to list of rows. No models, no schema. For a one-shot audit script, this was exactly the right call.

## The lesson

Heavy dependencies are the right tool when you need them — fuzzy matching at scale benefits from `rapidfuzz`, phone normalization really does want `phonenumbers`. But for a data quality audit that runs once or twice a week on a file you already have, the standard library gets you 80% of the value in a fraction of the setup time.

Before you `pip install`, ask whether `csv`, `re`, and `difflib` are already enough.

