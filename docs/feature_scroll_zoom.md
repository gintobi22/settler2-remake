# Feature: Scroll & Zoom (Step 7)

## Overview

This document describes the scroll & zoom system implemented in `settlers2_v12.jsx` as part of Step 7.

## Version

`const VERSION = 'v0.10'` — rendered as small grey text in the bottom-right canvas corner (always in screen space, after `ctx.restore()`).

## Viewport State Object

A module-level `viewport` object replaces the old `gameState.scrollX / scrollY`:

```javascript
const viewport = {
  offsetX: 0,   // horizontal scroll offset (world pixels)
  offsetY: 0,   // vertical scroll offset (world pixels)
  scale:   1.0, // zoom level
};
```

This is the **single source of truth** for camera position and zoom.

## Constants

```javascript
// Zoom limits
const ZOOM_MIN  = 0.25;
const ZOOM_MAX  = 2.0;
const ZOOM_STEP = 0.1;

// Edge scroll
const EDGE_SIZE  = 40;   // px from canvas edge that triggers scroll
const EDGE_SPEED = 8;    // world px per frame at maximum proximity

// Minimap
const MINIMAP_W      = 200;
const MINIMAP_H      = 100;
const MINIMAP_MARGIN = 12;
```

## Coordinate Conversion

```javascript
function screenToWorld(sx, sy, vp)  // screen px → world px (undoes scale + offset)
function worldToScreen(wx, wy, vp)  // world px → screen px (applies scale + offset)
function clampOffset(value, axis)   // clamps viewport offset to reasonable map bounds
```

## Rendering

The render loop wraps all draw calls in a `ctx.save / ctx.scale / ctx.translate / ctx.restore` block:

```javascript
ctx.clearRect(0, 0, canvas.width, canvas.height);
ctx.save();
ctx.scale(viewport.scale, viewport.scale);
ctx.translate(-viewport.offsetX, -viewport.offsetY);

// ... all existing draw calls (unchanged coordinate math) ...

ctx.restore();

// Minimap and VERSION string rendered in screen space after restore
drawMinimap(ctx, gameState, viewport);
ctx.fillText(VERSION, canvas.width - 6, canvas.height - 6);
```

## Input Handling

| Input | Action |
|-------|--------|
| Right-click drag | Pan the map |
| Middle-click drag | Pan the map (also supported) |
| Mouse wheel | Zoom in/out with cursor as pivot |
| Arrow keys | Pan by `SCROLL_STEP` world pixels |
| `+` / `=` | Zoom in centred on canvas |
| `-` | Zoom out centred on canvas |
| `Home` | Reset viewport to HQ at scale 1.0 |

## Edge Scrolling

Mouse proximity to canvas edges (within `EDGE_SIZE` = 40 px) triggers automatic scrolling via `applyEdgeScroll()`, called every game tick.

## Minimap

`drawMinimap(ctx, gameState, vp)` renders:
- A 200×100 pixel overview of terrain types in the bottom-right corner
- A white rectangle representing the current viewport

Clicking the minimap navigates the viewport to that map position.

## HQ Centring

`centreViewportOnHQ()` centres and resets the viewport (scale = 1.0) on the player HQ. Called automatically after any map load (WLD or procedural).

## Sprite LOD

- `shouldDrawSprites()` → `true` when `scale >= 0.5`
- `shouldDrawFigures()` → `true` when `scale >= 0.75`

These helpers can be used inside draw functions to skip sprites at very low zoom.

## Acceptance Criteria

- [x] `const VERSION = 'v0.10'` present near top of file
- [x] Version rendered as small grey text in bottom-right canvas corner
- [x] Right-click drag scrolls the map
- [x] Mouse wheel zooms with cursor position staying fixed on screen
- [x] Zoom range: 0.25× to 2.0×; scroll clamped to map bounds
- [x] Building placement, road drawing, and flag clicks work at all zoom levels
- [x] Minimap shows terrain colours and updates viewport rectangle in real time
- [x] Clicking the minimap navigates to that map position
- [x] Loading a WLD map resets viewport to centre on HQ at scale 1.0
- [x] Arrow keys scroll; +/– keys zoom; Home resets to HQ
- [x] Procedural map fallback still works (no regression)
