---
layout: post
title: "Why Your E2E Tests Break Every Time You Touch the UI"
date: 2026-04-07 01:42:32 +0000
categories: [debugging]
tags: [playwright, e2e, testing, selectors]
excerpt: "We shipped a full UI overhaul — dark mode, new loading states, restructured navigation — and woke up to 115 failing E2E tests. Every single one was a stale selector problem."
---

We spent a session doing good work: dark mode across 16 files, skeleton loaders replacing spinners, keyboard-accessible interactive cards, a new theme toggle. The UI got meaningfully better.

Then we checked the E2E suite. 115 tests failing.

Not because the features broke. Because the tests were written against an older version of the DOM, and now the DOM had changed. Classic stale selector problem.

## What a Stale Selector Actually Is

An E2E test drives the app through a browser the same way a user would — click here, type there, assert this text appears. To do that, it needs to *find* elements on the page. That's the selector.

The fragile way to find a button:

```js
// Breaks if you rename the CSS class, restructure the layout,
// or change the element type
page.locator('.btn-primary.action-card--clickable')
```

The resilient way:

```js
// Survives UI refactors — tied to meaning, not implementation
page.getByRole('button', { name: 'Add product' })
```

When we added dark mode variants, we renamed and reorganized CSS classes. When we swapped `text-gray-600` for `dark:text-gray-400`, selectors that were targeting style classes snapped. When we replaced raw `<div onClick>` with proper `<button>` elements for accessibility, tag-based selectors broke. When loading skeletons appeared before data, tests that expected immediate content failed the timing check.

None of these were *bugs*. They were improvements. But the tests were measuring the wrong thing.

## The Three Failure Patterns

Looking across the 115 failures, they fell into three buckets:

**1. Style-class selectors** — Tests querying `.dark\:text-white` or `.hover\:bg-gray-50` that evaporated when classes changed. These are the worst offenders because they're testing CSS, not behavior.

**2. Structural assumptions** — A test that finds a table row by `tr:nth-child(3)` breaks when you add a loading skeleton row above it, or when you reorder columns. The data is still there; the position isn't.

**3. Timing** — We added skeleton loaders, which means there's now a brief window where elements exist in the DOM but aren't populated yet. Tests that didn't wait for content to actually load started flaking immediately.

## What the Fixes Look Like

For style-class selectors, the fix is mechanical: replace with semantic queries. Playwright's `getByRole`, `getByLabel`, `getByText`, and `getByTestId` all survive visual refactors because they're anchored to meaning.

For structural selectors, the fix is usually adding a `data-testid` attribute to the element you care about. It's explicit, it survives restructuring, and it's zero-cost from a user perspective.

```html
<!-- Before: tests find this by position -->
<tr class="product-row">

<!-- After: tests find this by identity -->
<tr class="product-row" data-testid="product-row">
```

For timing issues, the fix is proper `waitFor` calls — not `waitFor(1000)` (a hardcoded sleep, which is the same as giving up), but `waitFor(() => expect(locator).toBeVisible())`. Let the test wait for the actual condition.

## The Uncomfortable Truth

The real problem is that we wrote tests *after* the UI was built, and we used whatever selectors were convenient. Convenient usually means "grab the most obvious CSS class or DOM structure." That works fine until you refactor — which is exactly when you most want your tests to be trustworthy.

The fix isn't just updating 115 selectors. It's changing how we write tests going forward: query by role and label first, fall back to `data-testid` for things without semantic identity, never query by style classes, and always wait for content rather than structure.

E2E tests are supposed to give you confidence when you change things. They can't do that job if every change breaks them.

