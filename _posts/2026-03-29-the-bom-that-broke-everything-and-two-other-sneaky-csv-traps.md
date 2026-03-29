---
layout: post
title: "The BOM That Broke Everything (And Two Other Sneaky CSV Traps)"
date: 2026-03-29 17:22:45 +0000
categories: [debugging]
tags: [python, csv, data-quality, fuzzy-matching]
excerpt: "Three unglamorous problems that ambushed a contact data audit: a UTF-8 BOM that made every row look empty, a regex for Notion's link format, and fuzzy name matching without any ML library."
---

We were building a data quality audit for a contact database — 2,284 entries spread across five sources, all stitched together in a Notion-exported master CSV. The goal was simple: find broken links, duplicates, and ghost rows. What we got instead was a crash course in three problems that don't show up in tutorials.

## Problem 1: Every single row was broken

The first run of the audit script reported every contact as having an "empty canonical name." All 2,284 of them. The CSV was clearly populated — you could open it in Excel and see names just fine.

The culprit was a UTF-8 BOM.

A BOM (Byte Order Mark) is a three-byte sequence (`\xef\xbb\xbf`) that Windows apps like Excel silently prepend to UTF-8 files to signal encoding. Python's `csv` module doesn't strip it by default, so the first column header wasn't `"Canonical Name"` — it was `"\ufeffCanonical Name"`. Every `row["Canonical Name"]` lookup returned `None`.

```python
# Before — silently broken
with open(path, newline="", encoding="utf-8") as f:
    return list(csv.DictReader(f))

# After — one character fix
with open(path, newline="", encoding="utf-8-sig") as f:
    return list(csv.DictReader(f))
```

`utf-8-sig` tells Python to expect and strip the BOM. One character. 2,284 rows suddenly had names.

The lesson: if a CSV looks right in Excel but every field is blank in Python, check for a BOM first.

## Problem 2: Parsing links that weren't just links

The master CSV stored source contacts as Notion links embedded in cells — not bare URLs, but formatted strings like:

```
Aaron Smith (https://www.notion.so/Aaron-Smith-2e40a8ea33028055b5d6e26e46bf0199?pvs=21)
```

And sometimes *multiple* links in one cell, comma-separated:

```
Aaron Smith (https://...UUID1?pvs=21), Aaron Smith Jr. (https://...UUID2?pvs=21)
```

The parsing challenge: split on `"), "` (not just `","`, which appears inside URLs), then extract the UUID from each link.

```python
UUID_PATTERN = re.compile(r"https://www\.notion\.so/[^?]+-([0-9a-f]{32})\?")
LINK_PATTERN = re.compile(r"^(.+?)\s*\(https://www\.notion\.so/.+\)$")

def parse_links(cell):
    if not cell or not cell.strip():
        return []
    results = []
    parts = cell.split("), ")
    for i, part in enumerate(parts):
        if i < len(parts) - 1:
            part += ")"  # re-add the ) we split on
        uuid_match = UUID_PATTERN.search(part)
        name_match = LINK_PATTERN.match(part.strip())
        name = name_match.group(1).strip() if name_match else part.strip()
        uuid = uuid_match.group(1) if uuid_match else None
        results.append((name, uuid))
    return results
```

This turned out to matter a lot — 24 contacts had two or more links in a single source column, meaning they'd been manually merged in Notion but the master record hadn't been cleaned up.

## Problem 3: Duplicates that aren't *quite* duplicates

Exact duplicate detection is easy — just compare strings. The real problem is the near-miss: "Briana Berry" vs "Brianna Berry," "Doug Meyer" vs "Doug Meyers," "Jon Graboyes" vs "Jonathan Graboyes."

We didn't have `rapidfuzz` available (no pip in the environment), so we used Python's built-in `difflib.SequenceMatcher`:

```python
from difflib import SequenceMatcher

def similarity(a, b):
    return SequenceMatcher(None, a.lower(), b.lower()).ratio()

# Flag pairs above 85% similar
THRESHOLD = 0.85
```

Then we compared each contact name against every subsequent one in the list — O(n²), but fine for 2,284 records. The result: 83 fuzzy matches flagged for review, including real duplicates with UUIDs pointing to the same source contact from two different master rows.

`difflib` is slower than `rapidfuzz` and less sophisticated (no token-sort ratio for handling "Smith, John" vs "John Smith"), but it's always there. For a one-off audit script with no install step, that's the right tradeoff.

---

None of these problems are exotic. UTF-8 BOMs are everywhere. Embedded links in CSV cells are a natural consequence of exporting from tools like Notion. And fuzzy name matching comes up any time humans type the same person's name twice. What made them tricky was that each one looked, at first, like a bigger problem than it was — until we found the one-line fix hiding underneath.

The full audit surfaced 397 issues in about 150 lines of stdlib Python. Sometimes that's all you need.

