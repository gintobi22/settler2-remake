# Rendering Gap Analysis — WebGL Terrain Renderer

## Overview

`settlers2_v12.jsx` currently renders all terrain using the Canvas 2D API. This document describes the architectural motivation and background for replacing terrain rendering with a WebGL layer (Step 5a).

---

## Current Architecture

All rendering is performed on a single `<canvas>` element using the Canvas 2D API:

1. **Background fill** — `ctx.fillRect` with a dark green colour covers the entire canvas.
2. **Terrain triangles** — Two triangles per map cell, drawn using `ctx.beginPath()` / `ctx.fill()` with 16×16 tiled `CanvasPattern` objects (decoded from inline TERR_HEX pixel data) or plain `fillStyle` colours.
3. **Overlays** — Territory borders, roads, trees, buildings, figures, and UI elements drawn on the same canvas.
4. **Viewport** — `ctx.save()` / `ctx.scale()` / `ctx.translate()` apply the camera transform to all world-space content.

### Identified Gaps

| Gap | Impact |
|-----|--------|
| All terrain drawn by the CPU each frame via Canvas 2D | High CPU load, poor frame rate on large WLD maps (e.g. 512×512 = 262 144 cells → ~524 288 triangles per frame) |
| No per-vertex shading | Flat-coloured terrain; no Gouraud lighting effect visible on hills and mountains |
| `fillRect` opaque background | Prevents compositing multiple canvas layers; the background must change colour to reveal terrain |
| Pattern tiling on CPU | Each `ctx.fill()` triggers a software texture lookup; no GPU texture sampling |

---

## Target Architecture

Two `<canvas>` elements CSS-stacked in the same container `div`:

```
Container div (position: relative)
  └─ WebGL canvas  (position: absolute; z-index: 0)  ← terrain only
  └─ Canvas 2D     (position: absolute; z-index: 1)  ← sprites, buildings, figures, UI
```

### Benefits

- **Performance**: Terrain geometry is uploaded once per map load as a `Float32Array` vertex buffer. Each frame only requires a single `gl.drawArrays` call; no per-triangle CPU work.
- **Gouraud shading**: Per-vertex `shade` attribute interpolated across triangles by the GPU, giving soft light/shadow transitions across hills.
- **Clean separation**: Sprite and game logic code remains entirely on Canvas 2D — zero impact on gameplay.
- **Scalability**: A 512×512 map produces ~3.1 M floats in one GPU upload; subsequent frames are GPU-bound only.

---

## Coordinate Systems

The existing isometric projection formula is preserved unchanged:

```
x = (col - row) × (TW / 2) + ox
y = (col + row) × (TH / 2) + oy − height × HEIGHT_FACTOR
```

Where `TW = 64`, `TH = 36`, `HEIGHT_FACTOR = 5`, `ox = canvasWidth/2 − (COLS−ROWS)×(TW/4)`, `oy = 30`.

Vertex positions in the WebGL buffer are pre-computed in world-space (without scroll). Scroll and zoom are applied uniformly in the vertex shader via `u_scroll` and `u_zoom` uniforms, ensuring pixel-perfect alignment between the WebGL terrain layer and the Canvas 2D sprite layer.

---

## Gouraud Shading Formula

Derived from settlers2.net and verified against Return to the Roots:

```
shade = 64 + 9×(P1−H) − 3×(P2−H) − 6×(P3−H) − 9×(P4−H)
clamped to [0, 128], normalised: shadeFloat = shade / 128
```

- `H`  = height of current node
- `P1` = upper neighbour (row−1, same col) — weighted +9 (light source direction)
- `P2` = right neighbour (same row, col+1) — weighted −3
- `P3` = lower-left neighbour (row+1, col−1) — weighted −6
- `P4` = lower neighbour (row+1, same col) — weighted −9

---

## Fallback Strategy

If `canvas.getContext('webgl')` returns `null` (e.g., hardware WebGL unavailable), the renderer falls back to the existing Canvas 2D terrain path with no change in visual output.

---

## Files Changed

| File | Change |
|------|--------|
| `settlers2_v12.jsx` | Dual-canvas DOM; WebGL context init; VBO build/upload; vertex/fragment shaders; Gouraud shade computation; `clearRect` on 2D canvas; Canvas 2D terrain code wrapped in `if(!glSt)` fallback |
| `docs/rendering_gap_analysis.md` | This document |
| `docs/webgl_terrain_spec_v2.md` | Full implementation specification |
