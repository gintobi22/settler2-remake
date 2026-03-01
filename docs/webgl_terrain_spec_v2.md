# WebGL Terrain Renderer — Implementation Spec v2

## Overview

Replace the existing Canvas 2D terrain rendering in `settlers2_v12.jsx` with a WebGL layer. All sprite/building/figure rendering stays on Canvas 2D unchanged.

---

## Architecture: Dual-Canvas Stack

```
Container div (position: relative; width: 100%; height: 100%)
  └─ WebGL canvas  (position: absolute; top:0; left:0; z-index: 0)
  └─ Canvas 2D     (position: absolute; top:0; left:0; z-index: 1)  ← existing ref
```

Rules:
- The existing Canvas 2D ref keeps its current name — zero changes to sprite rendering or mouse handlers
- The Canvas 2D must be cleared with `ctx.clearRect(...)` each frame (transparent), **not** a fill
- Both canvases have identical pixel dimensions
- All mouse events stay on the top (2D) canvas
- WebGL context options: `{ alpha: true, antialias: false, premultipliedAlpha: false }`
- If `canvas.getContext('webgl')` returns null: fall back gracefully to the old Canvas 2D terrain renderer

---

## Vertex Buffer Format

One interleaved `Float32Array`, 5 floats per vertex:

```
[x, y, u, v, shade]
```

- `x, y` — screen-space pixel position (pre-computed isometric projection + height offset, NOT scroll-compensated)
- `u, v` — normalised [0,1] UV into the terrain atlas texture
- `shade` — Gouraud brightness, 0.0=full shadow, 0.5=neutral, 1.0=bright

Each map cell produces 2 triangles × 3 vertices = 6 vertices × 5 floats = 30 floats.
Total VBO size = `width × height × 30` floats.

Triangle coverage matches the existing Canvas 2D renderer exactly:
- **t1** (RSU): `(col,row)` → `(col,row+1)` → `(col+1,row)`   — condition: `row < ROWS-1 && col < COLS-1`
- **t2** (LSD): `(col,row)` → `(col+1,row)` → `(col+1,row-1)` — condition: `col < COLS-1 && row > 0`

Edge cells that do not satisfy the condition use degenerate (zero-area) triangles.

---

## Gouraud Shading

Formula (from settlers2.net, verified against RttR):

```
shade = 64 + 9*(P1-H) - 3*(P2-H) - 6*(P3-H) - 9*(P4-H)
clamped to [0, 128], then normalised: shadeFloat = shade / 128.0
```

- `H`  = height of current node
- `P1` = upper neighbour `(row-1, col)` — weight +9
- `P2` = right neighbour `(row, col+1)` — weight −3
- `P3` = lower-left neighbour `(row+1, col-1)` — weight −6
- `P4` = lower neighbour `(row+1, col)` — weight −9

For procedural maps (no WLD block 13): compute from heightmap using the formula above.
For WLD maps: also computed from heightmap (block 13 shading not yet stored on nodes).

---

## Shaders

### Vertex Shader

```glsl
attribute vec2 a_position;
attribute vec2 a_texcoord;
attribute float a_shade;

uniform vec2 u_resolution;
uniform vec2 u_scroll;
uniform float u_zoom;

varying vec2 v_texcoord;
varying float v_shade;

void main() {
  vec2 pos = (a_position - u_scroll) * u_zoom;
  vec2 clipSpace = (pos / u_resolution) * 2.0 - 1.0;
  clipSpace.y *= -1.0;
  gl_Position = vec4(clipSpace, 0.0, 1.0);
  v_texcoord = a_texcoord;
  v_shade = a_shade;
}
```

Note: `u_resolution` is set to `vec2(canvas.width / 2, canvas.height / 2)`.

### Fragment Shader

```glsl
precision mediump float;

uniform sampler2D u_terrainTexture;

varying vec2 v_texcoord;
varying float v_shade;

void main() {
  vec4 texColor = texture2D(u_terrainTexture, v_texcoord);
  float brightness = clamp(v_shade * 2.0, 0.0, 1.5);
  gl_FragColor = vec4(texColor.rgb * brightness, texColor.a);
}
```

---

## Texture

**Current state**: TEX5.LBM is not yet decoded in this codebase. A placeholder 6×1 RGBA texture is created programmatically at initialisation time, with one representative colour per terrain type:

| Index | Terrain     | RGBA hex       |
|-------|-------------|----------------|
| 0     | GRASS       | `4b7b0fff`     |
| 1     | FOREST      | `739f1fff`     |
| 2     | MOUNTAIN    | `9f835bff`     |
| 3     | WATER       | `2971a6ff`     |
| 4     | SAND        | `c39f7fff`     |
| 5     | MEADOW      | `6a9a38ff`     |

UV coordinates: terrain type `t` → `u = (t + 0.5) / 6`, `v = 0.5`.

When TEX5.LBM is loaded: replace the placeholder with the full atlas and update UV mapping accordingly. Use `NEAREST` filtering and `CLAMP_TO_EDGE` wrapping. If textures appear vertically flipped: `gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true)` before `texImage2D`.

---

## Render Loop Integration

Each frame:
1. `gl.clear(gl.COLOR_BUFFER_BIT)` — clear WebGL canvas (background: S2 ocean blue `rgb(41,113,166)` = `(0.16,0.44,0.65,1.0)`)
2. Upload scroll/zoom uniforms: `u_scroll = [viewport.offsetX, viewport.offsetY]`, `u_zoom = viewport.scale`
3. `gl.drawArrays(gl.TRIANGLES, 0, vertexCount)`
4. `ctx.clearRect(0, 0, canvas.width, canvas.height)` — clear 2D canvas to transparent
5. All existing sprite/building/figure/UI draw calls (unchanged)

---

## Water Rendering

Water terrain type (T.WATER = 3) renders as a solid blue colour via the placeholder texture. The WebGL clear colour (`rgb(41,113,166)`) also fills any areas not covered by terrain geometry. Animated water is out of scope for this step.

---

## VBO Lifecycle

- **Build once per map load**: `vboDirtyRef.current = true` is set in `loadMap()` and during WebGL init.
- **Rebuild trigger**: checked at the start of each render frame; reset to `false` after upload.
- **VBO size**: `COLS × ROWS × 30` floats (includes degenerate triangles for edge cells).

---

## Fallback Behaviour

If `canvas.getContext('webgl')` returns `null`:
- `glStateRef.current` remains `null`
- The Canvas 2D terrain path is active (wrapped in `if(!glSt)` guard)
- `ctx.fillRect` background is used instead of `ctx.clearRect`
- All gameplay features work identically

---

## Acceptance Criteria

- [ ] Terrain renders via WebGL with correct colours matching terrain types
- [ ] Gouraud shading visible: mountains lighter on upper-left slopes, darker on lower-right
- [ ] All sprites (buildings, figures, flags, goods) render correctly on top of terrain
- [ ] No visible alignment gap between terrain and sprites
- [ ] Water areas render as solid blue (not transparent holes)
- [ ] Building placement works at zoom 0.5×, 1.0×, 2.0×
- [ ] Road drawing and flag clicks work at all zoom levels
- [ ] Carriers walk along roads at correct positions
- [ ] Both canvases stay in sync with viewport scroll/zoom; no drift
- [ ] If WebGL unavailable: graceful fallback to Canvas 2D terrain
- [ ] Ocean background colour is blue (not black/white)
- [ ] No console errors during normal gameplay
- [ ] Procedural map fallback still works

---

## Common Pitfalls

| Pitfall | Prevention |
|---------|------------|
| Y-axis flip | WebGL Y is up, Canvas Y is down. The vertex shader's `clipSpace.y *= -1.0` handles it. Do NOT also flip in the vertex buffer. |
| Texture vertically flipped | Add `gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true)` before `texImage2D` if terrain appears upside-down. |
| Sprite/terrain misalignment | Both canvases use identical `viewport.offsetX/Y/scale` values. |
| All-black terrain | Shade defaulting to 0.0 causes `0.0 * 2.0 = 0.0` in fragment shader = black. Always initialise to 0.5. |
| Canvas 2D hides terrain | The 2D canvas MUST use `clearRect`, not `fillRect`, when WebGL is active. |
| Mouse coordinates break | All mouse events stay on the top canvas (`cvRef`). `event.offsetX/Y` remains correct. |
