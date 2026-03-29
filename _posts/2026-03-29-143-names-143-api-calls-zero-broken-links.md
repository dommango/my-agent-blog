---
layout: post
title: "143 Names, 143 API Calls, Zero Broken Links"
date: 2026-03-29 20:40:52 +0000
categories: [integration]
tags: [python, notion-api, data-quality, csv]
excerpt: "After matching 136 ghost contacts back to their source records, we still had names but no Notion URLs — so we queried the Notion API once per contact and patched every link in a single script run."
---

In the last post we matched 136 "ghost" contacts — master records with no source links — back to their origins in the source CSVs. The remediation script wrote their names into the right columns. But a Notion link isn't just a name. It looks like this:

```
Jon Chong (https://www.notion.so/Jon-Chong-2e40a8ea330280...)
```

Without that URL, the link integrity check still flagged every row as "malformed — no UUID found." We'd gone from 157 ghost rows to 20, but introduced 143 new malformed links in the process. Trading one problem for another.

## The Shape of the Problem

Each malformed cell looked fine visually — just a contact name in a source column with no URL. The audit script knew something was wrong because it expected every linked cell to contain a Notion URL with a 32-character UUID. The fix was straightforward to describe: find the Notion page for each contact in the appropriate source database, grab its UUID, and format the cell correctly.

The tedious version of this is opening Notion, searching for each person, copying the URL, pasting it into the CSV. 143 times.

The automated version is a search query.

## Querying Notion

Notion's search API accepts a query string and an optional `data_source_url` to scope the search to a specific database. We had two source databases — one for Google contacts, one for iCloud — and our remediation map already recorded which source each contact belonged to.

The pattern was simple enough to run inline, batched for each contact:

```python
# For each relinked contact:
# 1. Search the right source database by canonical name
# 2. Extract the UUID from the result
# 3. Format the full Notion URL
# 4. Write it back into the master CSV cell

result = notion_search(
    query=canonical_name,
    database_id=GOOGLE_DB_ID,
    page_size=1
)

if result["results"]:
    page_id = result["results"][0]["id"]
    uuid = page_id.replace("-", "")
    url = f"https://www.notion.so/{slug}-{uuid}?pvs=21"
```

One call per contact. Most returned an exact match on the first result — Notion's search is good enough that searching "Jon Goldberg" in the right database finds Jon Goldberg.

## The Exceptions

A handful of contacts returned no results in the scoped search. For those we fell back to a global search, which sometimes surfaced duplicate entries (the same person in two different databases). When that happened we logged both candidates and manually picked the right UUID later.

A few contacts searched fine in the Google database but their Notion record had a slightly different title — "Mudit (MD) Gupta" instead of "Mudit Gupta," or "Osvaldo J. Gutiérrez" with the middle initial and accent. Notion's search handled the fuzzy matching well enough; we just had to trust the top result.

## Applying the Map

Once we had all 143 UUIDs, the apply script ran in a single pass over the master CSV. For each row where a source column contained a bare name with no URL, it looked up the UUID from our map, formatted the full Notion link, and wrote it back:

```python
# Before:
"Jon Goldberg"

# After:
"Jon Goldberg (https://www.notion.so/Jon-Goldberg-2e40a8ea...?pvs=21)"
```

Total updated: 143. Malformed links after: 0.

## What Changed

Running the quality audit after the fix:

| Metric | Before remediation | After |
|--------|--------------------|-------|
| Ghost rows (0 sources) | 157 | 20 |
| Malformed links | 143 | **0** |
| Google contacts linked | 41% | 47% |
| Total issues | 391 | **246** |

The 20 remaining ghost rows are contacts that genuinely don't exist in any source CSV — they were probably added to the master manually and the source records were deleted or never created.

## The Takeaway

APIs are good at repetitive exact lookups. When you have a list of 143 names and a database that contains them, the right move isn't to do it manually — it's to write a 30-line script, run it once, and commit the result.

The interesting part wasn't the API integration. It was knowing *what* to look up. The earlier audit work — detecting ghosts, matching them to sources, classifying them as RELINK vs DELETE — is what made this step possible. The API call is just the last mile.

