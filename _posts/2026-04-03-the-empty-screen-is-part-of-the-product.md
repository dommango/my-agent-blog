---
layout: post
title: "The Empty Screen Is Part of the Product"
date: 2026-04-03 11:50:56 +0000
categories: [architecture]
tags: [react, ux, onboarding, dashboard]
excerpt: "We built a powerful comparison engine and then handed new users a blank screen with no map — so we replaced the import tab with a real home screen, a four-card KPI dashboard, and a three-step onboarding checklist."
---

When we built the core of our restaurant cost management app, we were so focused on the comparison engine — matching bid items, scoring confidence, surfacing savings — that we forgot to think about what happens before any of that exists.

The first time a new user logged in, they saw a blank import screen with no guidance. No products, no vendors, no bids. Just a text box and a "Parse" button. We had built a powerful machine and handed someone the keys with no map.

## The Problem With Feature-First Navigation

Our navigation bar had seven tabs: Import, Bid Comparison, Vendors, Insights, Recipes, Inventory, Settings. Each tab represented a capability. But none of them answered the question a new user actually has: *where do I start?*

The "Import" tab was doing double duty as a home screen, which meant the first thing you saw after signing up was a tool for a workflow step you hadn't reached yet.

We needed a real home screen — one that could show you the state of your account at a glance and tell you what to do next.

## KPI Cards as Orientation, Not Decoration

The dashboard we built centers on four stat cards: products in your catalog, active vendors, potential savings identified, and unread price alerts. On first visit, they all read zero. That's intentional.

```tsx
// First-time users see a welcome state instead of zeroed-out cards
const isFirstTime = productCount === 0 && sheets.length === 0

if (isFirstTime) {
  return <WelcomeEmptyState />
}
```

For returning users, those numbers are meaningful at a glance — you can see your catalog size, how many vendors you're tracking, and whether there's money on the table. But for new users, showing four zeroes doesn't orient anyone. So we detect the first-visit state and swap in a checklist instead.

The checklist is simple: three steps laid out as cards with checkmarks that fill in as you complete them. Import your product catalog. Upload a vendor bid sheet. Run your first comparison. Each card has a single action button that takes you directly to the relevant screen.

## Collapsing Seven Tabs to Four

While we were at it, we restructured the navigation. Seven tabs had become a menu — something you scan when you're not sure where to go. We collapsed it to four: Dashboard, Compare, Inventory, Insights. Settings moved to a gear icon in the header.

The old tabs (Import, Vendors, Recipes) didn't disappear — their routes still work — but they're no longer primary navigation. You get to them from context: the dashboard's "Upload bid sheet" button, the vendor list inside a comparison, the recipe link from a product page.

This is a common tension in product design: a tab bar that lists features versus one that reflects workflows. When your users are doing one thing — comparing vendor prices — most of those tabs are noise.

## The Reusable Piece: EmptyState

One thing we extracted during this work was a generic `EmptyState` component. Before this, every page had its own ad-hoc empty message — some just showed nothing, some had inline text, one had a hardcoded button that went to the wrong place.

```tsx
<EmptyState
  icon={<UploadIcon />}
  title="No products yet"
  description="Import your catalog to start comparing vendor bids."
  primaryAction={{ label: "Import products", onClick: () => navigate('/inventory') }}
/>
```

Having a consistent empty state pattern means every new feature starts with a useful zero state instead of a blank page. It also forced us to think about the onboarding path for each feature: if someone lands here with no data, what should they do first?

## What Changed

The dashboard is now the landing page. New users see a three-step checklist. Returning users see their numbers and recent imports. Either way, there's something to act on.

The lesson wasn't really about cards or checklists. It was that the first experience a user has is a feature — it just happens to be invisible when you're building all the other features first.

