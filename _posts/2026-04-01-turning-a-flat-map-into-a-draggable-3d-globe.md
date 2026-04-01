---
layout: post
title: "Turning a Flat Map Into a Draggable 3D Globe"
date: 2026-04-01 04:01:46 +0000
categories: [architecture]
tags: [react, d3-geo, react-simple-maps, data-visualization]
excerpt: "Swapping a Mercator projection for an orthographic one in react-simple-maps — and wiring up mouse drag to rotate it — transformed a static travel map into something you actually want to play with."
---

The travel map on our portfolio site did its job: a flat world map with visited countries highlighted in gold and flight arcs sketched between cities. It was fine. But "fine" is not the vibe you want for something people are supposed to enjoy poking around on.

So we asked a simple question: what would it take to make it a globe?

## The flat map wasn't broken — it was just boring

The existing map used `react-simple-maps` with a Mercator projection. You've seen this: Greenland looks the size of Africa, Antarctica is a fat white stripe along the bottom, and the whole thing sits inert on the page like a screenshot. It worked for showing which countries we'd been to, but it had no life to it.

What we wanted was an orthographic projection — the classic "view from space" globe — that you could grab and spin to see the whole world.

## The projection swap is one line

Here's the thing about `react-simple-maps`: it wraps `d3-geo`, so switching projections is almost trivially easy. We went from:

```tsx
// flat Mercator — the default
<ComposableMap projection="geoMercator">
```

to:

```tsx
// globe view
<ComposableMap
  projection="geoOrthographic"
  projectionConfig={{ scale: 250, rotate: [rotation[0], rotation[1], 0] }}
>
```

The `rotate` array is the magic. D3's orthographic projection takes `[lambda, phi, gamma]` — longitude rotation, latitude tilt, and roll — and by updating those values on drag, you spin the globe.

We also added a `<Sphere>` element (also exported from `react-simple-maps`) to get the dark ocean background that makes visited countries really pop.

## Drag-to-rotate with vanilla React

There's no special library here. We tracked drag state with three React refs — `isDragging`, `lastMousePos`, and `rotation` — and wired them to `onMouseDown`, `onMouseMove`, and `onMouseUp` handlers on the map container.

```tsx
const handleMouseMove = useCallback((e: React.MouseEvent) => {
  if (!isDragging.current) return

  const dx = e.clientX - lastMousePos.current.x
  const dy = e.clientY - lastMousePos.current.y

  setRotation(prev => [
    prev[0] + dx * 0.4,   // drag left/right rotates longitude
    prev[1] - dy * 0.4,   // drag up/down tilts latitude
  ])

  lastMousePos.current = { x: e.clientX, y: e.clientY }
}, [])
```

The `0.4` multiplier is the sensitivity — lower feels sluggish, higher feels twitchy. We landed on `0.4` after a few seconds of manual testing.

## The visibility problem

Flat maps render every country whether it's in view or not. Globes don't work that way — a flight arc from Los Angeles to Tokyo should disappear when you rotate the Earth so both cities are on the back side.

For flight arcs, we check both endpoints before rendering:

```tsx
function isVisible(lon: number, lat: number, rotation: [number, number]) {
  // dot product of the point's 3D position with the viewing direction
  const lambda = (lon + rotation[0]) * (Math.PI / 180)
  const phi = (lat - rotation[1]) * (Math.PI / 180)
  return Math.cos(phi) * Math.cos(lambda) > 0
}
```

If either endpoint is behind the globe, we skip the arc entirely. Countries handle their own visibility through the orthographic projection clipping — d3-geo doesn't draw what it can't see.

## One less toggle

The old map had two toggle buttons: "Countries Visited" and "Flight History." In the redesign, we dropped the countries toggle. Visited countries are always shown — that's the whole point of the map. Simplifying to a single "Flight History" toggle cleaned up the UI nicely and removed a bit of state we didn't need.

## The payoff

Flat maps communicate data. Globes invite interaction. The moment the sphere appeared and we could drag it around, we wanted to keep spinning it. That's the difference between a data display and something that feels alive.

The entire change required no new dependencies — `d3-geo` was already installed transitively. The projection swap, the drag logic, and the visibility culling all fit comfortably in a single component file.

Sometimes the fun version really isn't that much harder to build.

