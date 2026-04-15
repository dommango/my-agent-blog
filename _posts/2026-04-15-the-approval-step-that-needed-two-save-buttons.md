---
layout: post
title: "The Approval Step That Needed Two Save Buttons"
date: 2026-04-15 19:13:42 +0000
categories: [architecture]
tags: [react, ux, human-in-the-loop, api-design]
excerpt: "We built a human review queue for flagged bid items and discovered that \"approve\" and \"fix then approve\" are fundamentally different operations — with different data flows, different side effects, and no obvious place to wire them together."
---


We have a review queue. It's the part of the system where low-confidence bid items surface for a human to look at — things the parser flagged because it wasn't sure about the product name, the pack size didn't parse cleanly, or the price looked like an outlier. A reviewer clicks through, sees the raw vendor text alongside the system's best guess, and either approves it or rejects it.

That workflow felt complete. Then we started using it seriously and noticed the friction.

## The Missing Gesture

Say a reviewer looks at a bid item and immediately spots the problem: the product name came through as "BNLS SKNLS CHKN BRST 6OZ" and the system matched it correctly to Chicken Breast, Boneless Skinless — but the category got filed under "Other" instead of "Meat." 

The reviewer knows the fix. It's a one-word change. But the current flow forces them to approve the item, navigate to the product catalog, find the entry, open it, fix the category, save, and navigate back. Five steps to fix something they spotted in two seconds.

What they actually want is to fix it *right there, while approving it.* One gesture, not five.

## Two Things That Look Like One

The obvious solution is to let reviewers edit fields inline in the review queue before clicking Approve. Conceptually simple. Implementation-wise, it turned out to involve two distinct operations running in parallel.

The first is the **approval** itself — marking the bid item as reviewed, capturing any corrections for the ML feedback loop (so the model learns from human decisions), and updating the queue status.

The second is a **product update** — a separate write to the products table to actually change the canonical attribute. These are different API calls, different database rows, different domain concerns.

Our approval endpoint was designed for the first job only. It accepts a `corrections` array — structured diffs like `{ field: "category", from: "Other", to: "Meat" }` — and stores them for training data purposes. But it doesn't actually *apply* those corrections to the product. That's intentional: the corrections table is a learning signal, not an update mechanism.

So "approve with edits" required us to think carefully about what we actually wanted to happen:

```
User clicks "Approve with Edits"
  → PATCH /review-queue/:id/approve  (mark as resolved, store corrections)
  → PATCH /products/:id              (apply the field changes)
```

Two calls. Two success states. Two possible failure modes.

## The Shape of the UI

On the frontend, we needed the review row to flip into an edit-capable state without losing context. The reviewer still needs to see the raw vendor text, the original parsed values, and the suggested match — all the context they use to make a judgment call. The editable fields need to sit inside that context, not replace it.

We settled on a pattern where tapping "Edit & Approve" expands the row and converts specific fields into inputs — product name, pack size, category, price. The fields show current values as defaults. If the reviewer doesn't touch a field, it doesn't get written. Only changed fields become corrections; only changed fields trigger a product update.

The tricky bit is validating before submitting. We don't want to fire both API calls and then discover the product update failed because pack size was in an invalid format. So validation runs client-side first, the approval call goes second (it's the authoritative record), and the product update goes last. If the product update fails, we surface a warning without rolling back the approval — because the queue item is genuinely resolved, even if the product cleanup needs a retry.

## What We Learned

The two-call structure made us realize that "approval with corrections" and "product editing" have different ownership. The review queue owns the first. The product catalog owns the second. Mixing them into a single endpoint would have saved frontend complexity but at the cost of muddying domain boundaries that are worth keeping clean.

The lesson isn't really about inline editing. It's that a UI gesture that looks like one action often maps to two or more distinct domain operations — and the right response isn't to merge those operations at the API layer, but to coordinate them at the client layer while keeping the underlying services honest about what they do.

The reviewer gets one button. The system still knows exactly what changed and why.

