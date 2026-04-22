---
layout: post
title: "Store the Raw Text. Add the Copy Button."
date: 2026-04-22 03:31:38 +0000
categories: [til]
tags: [ux, clipboard, parsing, react]
excerpt: "When a document parser gets something wrong, the fastest recovery path is letting users grab the original text and start over — and a single copy-to-clipboard button is all it takes."
---

Every document processing pipeline makes a bet: that the parser gets it right. Ours handles vendor price lists — dense PDFs and spreadsheets full of abbreviations, inconsistent formatting, and handwritten-looking line items. We parse each one, extract structured data, and store it in the database.

Most of the time, the bet pays off. When it doesn't, users are stuck.

## The Problem With Parsed-Only Views

Once a bid sheet is imported and processed, the UI shows structured rows: product name, pack size, price per case. Clean, sortable, reviewable. But if something looks off — a price that's obviously wrong, a product name that got mangled — the user's first question is always the same: *what did the original document actually say?*

We had no good answer. We'd tell them to re-upload the file. Which meant going to find the file. Which they may not have bookmarked. Which they may have gotten over email weeks ago.

So we started storing the raw text — the full extracted content of the document before any parsing logic touches it — in a `raw_text` column alongside the parsed bid sheet. Then we had to decide how to surface it.

## The `<details>` Element Was the Right Call

The raw text panel on the bid sheet detail page is a native HTML `<details>` element:

```html
<details className="rounded-lg border border-gray-200 dark:border-gray-700">
  <summary className="cursor-pointer px-4 py-2 text-sm font-medium">
    Source text ({charCount} chars)
  </summary>
  <pre className="max-h-96 overflow-y-auto px-4 py-3 text-xs font-mono">
    {rawText}
  </pre>
</details>
```

No `useState`. No click handler. No animation library. The browser handles the toggle natively, it's keyboard accessible by default, and it collapsed-by-default behavior fits perfectly — most users never need it, but it's one click away when they do.

The panel only renders when `rawText` has content, so it's invisible for older records that predate the column.

## Adding the Copy Button

With the raw text visible, the next obvious frustration is: you can't easily *use* it. Selecting text inside a scrollable `<pre>` block across hundreds of lines is tedious. So we added a copy button in the `<summary>` bar:

```tsx
const [copied, setCopied] = useState(false)

async function handleCopy(e: React.MouseEvent) {
  e.preventDefault() // Don't toggle the <details> open/closed
  await navigator.clipboard.writeText(rawText)
  setCopied(true)
  setTimeout(() => setCopied(false), 2000)
}

// In the summary:
<button onClick={handleCopy} className="ml-auto text-xs text-gray-500 hover:text-gray-800">
  {copied ? 'Copied!' : 'Copy'}
</button>
```

The `e.preventDefault()` is the subtle part. Clicking inside a `<summary>` element normally toggles the parent `<details>` open or closed. Without it, hitting "Copy" would also collapse the panel — confusing enough to feel broken.

The two-second "Copied!" flash gives just enough feedback to confirm success without being annoying. No toast, no modal, no animation.

## What People Actually Do With It

The use cases turned out to be broader than we expected:

- **Re-processing with different settings**: paste the raw text into the manual import flow, tweak the vendor or date, and re-run the parser
- **Debugging parser behavior**: paste into a text editor to search for exactly what the parser should have found
- **Sharing with a vendor**: the raw extracted text is often cleaner than the PDF for email — paste it directly into a reply asking for a correction
- **Backup re-paste**: if a sheet was uploaded from a one-time email attachment and the file is gone, the stored raw text is now the canonical recoverable copy

## The Deeper Habit

This feature took about fifteen minutes to build, but it reflects a habit that pays off consistently: **treat the original source as a first-class artifact**.

In document processing, it's tempting to think of the parsed output as the real data and the source as a temporary input. But every parser has a failure mode, and when it fails, the source is the only ground truth you have. If you haven't stored it, you're asking users to repeat work they've already done.

Store the raw text. Surface it somewhere unobtrusive. Add the copy button. When something goes wrong — and it will — you'll have given users a one-click way back to the beginning.
