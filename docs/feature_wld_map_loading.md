# Feature Spec: Original .WLD Map Loading (Step 5)

**Status:** Ready for implementation  
**Priority:** High — prerequisite for meaningful gameplay testing  
**Estimated effort:** Medium (2–3 sessions)  
**Depends on:** Step 7 (Scroll & Zoom) must follow immediately after, as WLD maps are too large for the current fixed viewport  

---

## 1. Overview

Replace the current Perlin-noise procedural map generator with a loader for original Settlers II `.WLD` / `.SWD` binary map files. This gives the game authentic terrain, correct resource distribution (ore only in mountains, fish only near water, etc.), pre-set player HQ positions, and properly sized maps.

The loader runs entirely in the browser — the user selects a `.WLD` file via a file picker, it is parsed client-side from an `ArrayBuffer`, and the resulting map state object replaces the procedural one with zero server involvement.

---

## 2. File Format Reference

Source: settlers2.net technical documentation + `s25client/libs/s25main/world/MapLoader.cpp`

### 2.1 Header (2352 bytes total)

| Offset | Size | Content |
|--------|------|---------|
| 0 | 10 bytes | File ID: ASCII `"WORLD_V1.0"` |
| 10 | variable | Map title (null-terminated, typically 19–23 bytes) |
| after title | 1 byte | Terrain type: `0`=Greenland, `1`=Wasteland, `2`=Winter |
| +1 | 1 byte | Player count (1–7 stored; max 4 in original gameplay) |
| +2 | 14 bytes | 7 × HQ positions as `(uint16 x, uint16 y)` pairs |
| +16 | 2250 bytes | Passable area data (250 land/water masses × 9 bytes each) |

> **Implementation note:** Read the header as a `DataView` over the `ArrayBuffer`. The file ID check (`"WORLD_V1.0"`) should be the first validation — throw a user-friendly error if it fails.

### 2.2 Map Dimensions

Immediately after the 2352-byte header, before the data blocks, are two `uint16` values:

```
mapWidth  = dataView.getUint16(offset, true)   // little-endian
mapHeight = dataView.getUint16(offset+2, true)
```

Valid sizes: 64×64, 80×80, 96×96 ... up to 256×256 (always multiples of 16).

### 2.3 Data Blocks (14 blocks)

Each block is preceded by a **16-byte sub-header**:

| Sub-header offset | Content |
|-------------------|---------|
| 0 | Block type identifier (uint16) |
| 2 | Unknown (skip) |
| 4 | Block data length in bytes (uint32) |
| 8–15 | Padding / unknown |

Block data length = `mapWidth × mapHeight` bytes (one byte per node).

**The 14 blocks in order:**

| # | Name | Key usage |
|---|------|-----------|
| 1 | **Height map** | Altitude per node. Values: base 0/10/40, max 60. Max diff between adjacent nodes = 5. Multiply by `HEIGHT_FACTOR=5` for pixel offset. |
| 2 | **Texture RSU** | Terrain type for upper (right-side-up) triangles. See terrain table below. |
| 3 | **Texture LSD** | Terrain type for lower (upside-down) triangles. |
| 4 | Roads | Pre-placed road bits — skip for now (player builds their own) |
| 5 | **Object Index** | `0xC4–0xC6` = tree, `0xCC–0xCE` = granite, others = decorations |
| 6 | **Object Type** | Bit flags qualifying object in block 5 (tree species, size, etc.) |
| 7 | Animals | Spawn points — skip for now |
| 8 | Unknown | Skip |
| 9 | **Build Sites** | `0`=none, `1`=flag only, `2`=hut, `3`=house, `4`=castle, `5`=mine |
| 10 | Unknown | Skip (filled with 7s) |
| 11 | Editor cursor | Skip |
| 12 | **Resources** | Underground resources — see resource byte format below |
| 13 | Gouraud shading | Lighting values — skip initially |
| 14 | Passable areas | Area indices 0–249, `254`=impassable — skip initially |

> **Implementation note:** Even skipped blocks must be stepped over correctly using the sub-header's length field. Do not hardcode offsets.

---

## 3. Terrain Types (Greenland — Terrain Type 0)

Used for blocks 2 and 3. The lower 6 bits are the terrain ID; bit 6 (`0x40`) flags a harbor-capable coastline.

| ID | Terrain | Buildable | Walkable | Mineable | Map to existing type |
|----|---------|-----------|----------|----------|----------------------|
| 0 | Savannah | ✅ | ✅ | ❌ | `'grass'` |
| 1 | Mountain 1 | ❌ | ✅ | ✅ | `'mountain'` |
| 2 | Snow | ❌ | ❌ | ❌ | `'snow'` |
| 3 | Swamp | ❌ | ❌ | ❌ | `'water'` |
| 4 | Desert variant | ✅ | ✅ | ❌ | `'grass'` |
| 5 | Water | ❌ | ❌ | ❌ | `'water'` |
| 6 | Buildable Water | ❌ | ❌ | ❌ | `'water'` |
| 7 | Desert variant 2 | ❌ | ✅ | ❌ | `'grass'` |
| 8 | Meadow 1 | ✅ | ✅ | ❌ | `'grass'` |
| 9 | Meadow 2 | ✅ | ✅ | ❌ | `'grass'` |
| 10 | Meadow 3 | ✅ | ✅ | ❌ | `'grass'` |
| 11 | Mountain 2 | ❌ | ✅ | ✅ | `'mountain'` |
| 12 | Mountain 3 | ❌ | ✅ | ✅ | `'mountain'` |
| 13 | Mountain 4 | ❌ | ✅ | ✅ | `'mountain'` |
| 14 | Steppe | ✅ | ✅ | ❌ | `'grass'` |
| 15 | Flower Meadow | ✅ | ✅ | ❌ | `'grass'` |
| 16 | Lava | ❌ | ❌ | ❌ | `'water'` |
| 17 | Silver (color) | ❌ | ✅ | ❌ | `'mountain'` |
| 18 | Mountain Meadow | ✅ | ✅ | ❌ | `'grass'` |
| 19 | Water variant | ❌ | ❌ | ❌ | `'water'` |
| 22 | Lava variant | ❌ | ❌ | ❌ | `'water'` |
| 34 | Mountain Meadow 2 | ❌ | ✅ | ✅ | `'mountain'` |

**Extract terrain ID:** `terrainId = textureByte & 0x3F`  
**Extract harbor flag:** `isHarbor = (textureByte & 0x40) !== 0`

---

## 4. Resource Byte Format (Block 12)

One byte per node. Decode as follows:

```javascript
function decodeResource(byte) {
  if (byte === 0x21) return { type: 'water',    amount: 1 };
  if (byte === 0x87) return { type: 'fish',     amount: 1 };
  const amount = byte & 0x07;  // bottom 3 bits = quantity (0–7)
  const kind   = byte & 0xF8;  // top 5 bits = resource kind
  if (kind === 0x40) return { type: 'coal',     amount };
  if (kind === 0x48) return { type: 'ironOre',  amount };
  if (kind === 0x50) return { type: 'gold',     amount };
  if (kind === 0x58) return { type: 'granite',  amount };
  return null; // no resource
}
```

Store the decoded resource on each map node. This is what the geologist reveals later and what mines deplete.

---

## 5. Object Index / Type Decoding (Blocks 5 & 6)

```javascript
function decodeObject(indexByte, typeByte) {
  if (indexByte >= 0xC4 && indexByte <= 0xC6) {
    // Tree — species encoded in typeByte
    const species = ['pine','birch','oak','palm','pine2','fir','cypress','cherry'][typeByte & 0x07];
    const size    = (typeByte >> 3) & 0x07; // 0=sapling, 5=full grown
    return { kind: 'tree', species, size };
  }
  if (indexByte >= 0xCC && indexByte <= 0xCE) {
    // Granite
    const size = indexByte - 0xCC; // 0=small, 1=medium, 2=large
    return { kind: 'granite', size };
  }
  // Other decoration objects — ignore for now
  return null;
}
```

---

## 6. Build Quality (BQ) from Block 9

The map file pre-computes valid build sites. Map directly to the existing BQ system:

```javascript
const BQ_MAP = {
  0: 0,  // no build
  1: 1,  // flag only
  2: 2,  // hut
  3: 3,  // house
  4: 4,  // castle
  5: 5,  // mine (on mountain terrain)
};
```

Use block 9 values as the authoritative `buildQuality` on each node, **overriding** any BQ computed from terrain heuristics. This is more accurate than deriving BQ from terrain type alone.

---

## 7. HQ Placement

The header contains up to 7 HQ positions. For a single-player game (player 0):

```javascript
const hqX = dataView.getUint16(hqOffset + 0, true);
const hqY = dataView.getUint16(hqOffset + 2, true);
```

- Place the player's HQ at position `(hqX, hqY)` with the standard Roman HQ sprite
- Initialise the HQ inventory using the **Normal** starting goods table (see reference doc section 9)
- If any HQ position is `(0xFFFF, 0xFFFF)` or `(0, 0)`, that player slot is unused — skip it

---

## 8. Map State Object

The parser should return a map state object compatible with the existing game state structure, plus new fields:

```javascript
{
  // Existing fields (same structure as procedural map)
  width:    Number,         // from file
  height:   Number,         // from file
  nodes: [                  // flat array, index = y * width + x
    {
      height:      Number,  // 0–60, from block 1
      terrainRSU:  String,  // 'grass'|'mountain'|'water'|'snow', from block 2
      terrainLSD:  String,  // same, from block 3
      buildQuality:Number,  // 0–5, from block 9
      object:      Object|null, // { kind, species, size } from blocks 5+6
      resource:    Object|null, // { type, amount } from block 12
      harbor:      Boolean, // from texture bit 0x40
    }
  ],

  // New fields
  terrainType: Number,      // 0=Greenland, 1=Wasteland, 2=Winter
  title:       String,      // map title from header
  playerCount: Number,      // from header
  hqPositions: [            // array of {x, y} per player slot
    { x: Number, y: Number }
  ],
}
```

---

## 9. UI: File Picker

Add a **"Load Map"** button to the game toolbar (or a startup screen if none exists yet). On click:

```javascript
const input = document.createElement('input');
input.type = 'file';
input.accept = '.wld,.swd,.WLD,.SWD';
input.onchange = (e) => {
  const file = e.target.files[0];
  const reader = new FileReader();
  reader.onload = (ev) => {
    try {
      const map = parseWLDMap(ev.target.result); // ArrayBuffer → map state
      loadMap(map);   // replace current map state, reset buildings/figures/goods
    } catch (err) {
      alert(`Failed to load map: ${err.message}`);
    }
  };
  reader.readAsArrayBuffer(file);
};
input.click();
```

When a map is loaded:
- Clear all existing buildings, flags, roads, figures, goods
- Set map dimensions, nodes, and terrain from parsed data
- Place player HQ at `hqPositions[0]` with Normal starting inventory
- Reset viewport to centre on the HQ (scroll/zoom will be needed for large maps)
- Retain all production/building logic — it does not change

---

## 10. Parser Function Skeleton

```javascript
function parseWLDMap(arrayBuffer) {
  const dv = new DataView(arrayBuffer);
  let offset = 0;

  // 1. Validate file ID
  const fileId = String.fromCharCode(...new Uint8Array(arrayBuffer, 0, 10));
  if (fileId !== 'WORLD_V1.0') throw new Error('Not a valid Settlers II map file');
  offset = 10;

  // 2. Read title (null-terminated)
  let title = '';
  while (dv.getUint8(offset) !== 0) title += String.fromCharCode(dv.getUint8(offset++));
  offset++; // skip null terminator

  // 3. Terrain type + player count
  const terrainType = dv.getUint8(offset++);
  const playerCount = dv.getUint8(offset++);

  // 4. HQ positions (7 slots × 4 bytes each)
  const hqPositions = [];
  for (let i = 0; i < 7; i++) {
    const x = dv.getUint16(offset, true);
    const y = dv.getUint16(offset + 2, true);
    offset += 4;
    if (x !== 0xFFFF && y !== 0xFFFF) hqPositions.push({ x, y });
  }

  // 5. Skip rest of header to byte 2352
  offset = 2352;

  // 6. Map dimensions
  const width  = dv.getUint16(offset, true); offset += 2;
  const height = dv.getUint16(offset, true); offset += 4; // +2 unknown

  const nodeCount = width * height;
  const blocks = [];

  // 7. Read all 14 blocks
  for (let b = 0; b < 14; b++) {
    // 16-byte sub-header
    const blockType = dv.getUint16(offset, true);
    const blockLen  = dv.getUint32(offset + 4, true);
    offset += 16;
    blocks.push(new Uint8Array(arrayBuffer, offset, blockLen));
    offset += blockLen;
  }

  // 8. Build node array
  const nodes = [];
  for (let i = 0; i < nodeCount; i++) {
    const texRSU = blocks[1][i];
    const texLSD = blocks[2][i];
    nodes.push({
      height:       blocks[0][i],
      terrainRSU:   mapTerrain(texRSU & 0x3F),
      terrainLSD:   mapTerrain(texLSD & 0x3F),
      harbor:       !!(texRSU & 0x40) || !!(texLSD & 0x40),
      buildQuality: BQ_MAP[blocks[8][i]] ?? 0,
      object:       decodeObject(blocks[4][i], blocks[5][i]),
      resource:     decodeResource(blocks[11][i]),
    });
  }

  return { width, height, nodes, terrainType, title, playerCount, hqPositions };
}
```

---

## 11. Viewport Reset on Map Load

After loading, centre the viewport on the player HQ. Since scroll/zoom (Step 7) will be implemented immediately after, stub this as:

```javascript
function centreOnHQ(hqX, hqY) {
  // placeholder — Step 7 will implement full scroll/zoom
  // For now, offset the canvas draw origin so HQ is near centre
  gameState.viewOffset = {
    x: hqX * TR_W - canvasWidth  / 2,
    y: hqY * TR_H - canvasHeight / 2,
  };
}
```

---

## 12. Test Maps

The following original campaign maps are good test cases (shipped with Settlers II / RttR):

| Map | Size | Notes |
|-----|------|-------|
| `MISS0001.WLD` | 80×80 | Tutorial — simple terrain, small |
| `MISS0002.WLD` | 128×128 | First real campaign map |
| `GREEN01.WLD` | 176×176 | Standard multiplayer map |

If the user does not have the original game files, RttR ships free maps under GPL at:  
`github.com/Return-To-The-Roots/s25client/tree/master/DATA/MAPS`

---

## 13. Out of Scope for This Step

- Wasteland / Winter terrain textures (TEX6/7.LBM) — Greenland only for now
- Pre-placed roads from block 4 — player builds their own
- Animal spawning from block 7
- Gouraud shading from block 13
- Fog of war / passable area masks from block 14
- Multiplayer HQ positions (player 0 only)

These are noted in the development plan as later polish steps.

---

## 14. Acceptance Criteria

- [ ] `.WLD` file can be selected via browser file picker
- [ ] Invalid files produce a clear error message
- [ ] Map dimensions, height, terrain, BQ, objects, and resources are parsed correctly
- [ ] Player HQ spawns at the correct map position with Normal starting inventory
- [ ] All existing production logic (carriers, goods, buildings) continues to work unchanged
- [ ] Woodcutters can only be placed where trees exist on the WLD map
- [ ] Mines can only be placed on mountain terrain nodes (BQ=5)
- [ ] Quarries work on nodes with granite objects
- [ ] No regression in procedural map mode (keep it as a fallback option)
