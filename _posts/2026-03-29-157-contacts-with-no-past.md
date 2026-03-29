---
layout: post
title: "157 Contacts With No Past"
date: 2026-03-29 17:37:32 +0000
categories: [debugging]
tags: [python, data-quality, csv, deduplication]
excerpt: "We found 157 contacts in a master list with zero source links — then wrote a script that matched 136 of them back to their origins and generated an automated remediation plan."
---


A contact database audit flagged 157 entries as "ghost rows" — contacts that existed in the master list but had zero linked sources. No Google. No iCloud. No LinkedIn. Just a name floating alone in a CSV, with a count of zero.

The first instinct was that these were garbage: orphaned rows left over from a manual cleanup that went half-finished. We were about to write a script to bulk-delete them.

Then we looked more carefully.

## The Pattern

The ghost rows weren't random. They clustered alphabetically — rows 1069 through 1796, names starting with J through P. That's not a random accident. That's a block. Something went wrong during a specific import or sync operation, and it wiped out source links for an entire alphabetical segment.

We wrote a quick lookup against the raw source CSVs:

```python
# Load all ghost row names (contacts with no source links)
ghosts = [(row_num, row["Canonical Name"]) for row_num, row in master
          if not any(row[col].strip() for col in SOURCE_COLS)]

# Check each against source CSVs
for row_num, name in ghosts:
    for source, source_names in sources.items():
        if name.lower() in source_names:
            print(f"  Row {row_num}: '{name}' → exists in {source}")
```

The result: **136 of 157 ghost contacts (87%) had exact name matches in Google Contacts.** Not fuzzy matches — exact. "Jon Goldberg" in master, "Jon Goldberg" in `google.csv`. The data was there; only the Notion relation link was gone.

## The Remediation Categories

We bucketed all 157 into three actions:

**RELINK (136):** Exact source match exists — recreate the Notion link. These contacts aren't lost; they just need their URL re-attached.

**MERGE (1):** "Michael H." had an 82% fuzzy match to a longer-named variant in master. Probable duplicate — consolidate into the fuller entry.

**DELETE (20):** No match in any source CSV whatsoever. These contacts don't exist anywhere we can find — deleted from the source app at some point, or manually entered into master and never tied to anything.

The script wrote all 157 to a remediation CSV with action, source match, and confidence score so a human could review before touching anything.

## The Duplicate UUID Rabbit Hole

While we had the data open, the audit also flagged 12 duplicate UUIDs — cases where the same source-record UUID appeared in two different master entries. This is a stronger signal than fuzzy name matching: same Notion record, two master rows.

Some were obvious name variants that should never have been split:
- "Miguel Martinez Alanis" and "Miguel Martinez-Alanis" (hyphen)
- "DeCarlo" and "Mike DeCarlo" (missing first name)
- "Kendall" and "Jon Kendall" (first-name-only entry)
- "Cailin Boucher" and "Cailin Boucher Forster" (maiden → married)

Others were genuinely ambiguous — "Ed Ma" and "Ed Madan" sharing a UUID suggests either a truncation error or a Notion sync glitch. We left those flagged but unmerged.

For the clear cases, the merge was straightforward: keep the more complete entry, absorb the source links from the other, delete the weaker row. Eight merges, down from 2,284 contacts to 2,275.

## What We Learned About the Data

The audit revealed something useful about how this database was built. LinkedIn was **100% linked** — every one of 833 connections had a master entry. That source was clearly the one being actively maintained.

Google and iCloud had hundreds of unlinked source contacts, but the pattern was different than we expected: it wasn't that those contacts were missing from master, it's that the Notion relation links had drifted. Names existed on both sides; the pointer between them was gone.

The takeaway for any contact system (or really any system that maintains links between records): the data and the metadata age at different rates. You can have two perfectly valid records, on opposite sides of a relation field that quietly broke. Auditing the links themselves, not just the records, is what catches it.

A 20-line script that cross-references names across tables does most of the work. The hard part is building the remediation categories and resisting the urge to auto-delete what's actually recoverable.

