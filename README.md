# World Map

Two versions, both in this folder. Double-click a `.bat` file to open.

---

## Version 1: `index.html` - Full version

Built on **Leaflet**. Most features.

### Features

| Feature | Description |
|---|---|
| **Search** | Type a place name, live suggestions, Enter to jump |
| **5 base layers** | Street / Amap / Esri Street / Amap Satellite / Esri Satellite |
| **Drag to pan** | Move anywhere in the world |
| **Scroll to zoom** | Levels 2 - 19 |
| **Locate** | Uses browser GPS, drops a marker |
| **Coordinates** | Live lat/lon under the cursor |

### Language

- **Search**: works with any language input
- **Map labels**:
  - `Street` (OSM) - mixed English and local names
  - `Amap` - Chinese labels
  - `Esri Street` - English labels

### Downside

Loads Leaflet from a CDN (~150 KB). First open takes a few seconds.

---

## Version 2: `static.html` - Zero dependency

**No external JS libraries.** The whole map engine is in this one file.

### Differences

| | Full version | Static version |
|---|---|---|
| External deps | Leaflet (CDN) | **None** |
| Search | Yes | No |
| Base layers | 5 | 5 |
| Drag / zoom | Yes | Yes |
| Touch gestures | Yes | Yes (pinch) |
| File size | ~17 KB | ~26 KB |
| Works offline | No (needs Leaflet) | **Yes** (only tiles need network) |

### Why this version exists

1. **CDN outages don't matter** - unpkg and jsdelivr are unreliable in some regions
2. **Opens offline** - the page itself needs zero network, only map tiles do
3. **Easy to modify** - no framework, plain code
4. **Educational** - shows exactly how tile maps are assembled

### How it works

A minimal tile map engine written from scratch:

- **Web Mercator projection** - lat/lon to world pixel coordinates and back
- **Tile scheduling** - computes which tiles the viewport needs, adds and removes them
- **Sub-level zoom** - tiles fetched at integer zoom, one CSS transform on the container for smoothness
- **Seamless stitching** - no per-tile scaling, so no sub-pixel gaps

### Performance design

Zooming used to stutter because every level change tore down all tiles and
rebuilt them one by one, each with its own `transform`.

The current design avoids that:

- **One transform for everything.** Tiles are plain `<img>` elements positioned
  in the active level's pixel space. The whole tile container gets a single
  `translate3d + scale` per frame, so the GPU composites it in one pass.
- **Old level kept as a backdrop.** When zoom crosses an integer level, the new
  level is built on top while the previous one stays visible underneath. Nothing
  is removed until the new level is ~85% loaded, so there is no white flash.
- **Wheel deltas are batched.** Rapid wheel events accumulate and are applied
  once per animation frame instead of triggering a render each time.
- **No rebuild on pan.** A one-tile padding ring is kept around the viewport, so
  small pans reuse what is already loaded.
- **Tiles fade in.** Each tile fades over 180ms when it arrives instead of
  popping in.

Measured on this machine (headless Chrome, 1482x748):

- 6 consecutive zoom levels: **182 ms total**, ~30 ms per level
- 20 rapid wheel events: **16.7 ms average frame**, **17 ms worst frame**,
  **0 frames over 50 ms** - a steady 60 fps
- After settling, only the active level remains (54 tiles at z10, old level freed)

A debug handle is exposed for inspection:

```js
window.__map.stats()   // { tiles, pending, activeZ, prevZ, zoom }
```

---

## Data sources

All tested working, **no API key required**:

| Layer | Source | Notes |
|---|---|---|
| Street | `tile.openstreetmap.de` | OSM German mirror, good label coverage |
| Amap | `webrd0x.is.autonavi.com` | Chinese labels, fast in China |
| Esri Street | `server.arcgisonline.com` | English, global coverage |
| Amap Satellite | `webst0x.is.autonavi.com` | Satellite imagery |
| Esri Satellite | `server.arcgisonline.com` | Satellite imagery, global |

Search uses **Photon** (`photon.komoot.io`), OpenStreetMap based, free and unlimited.

---

## Known issues

**Esri tiles occasionally time out.** If a layer stays blank, switch to another one - the two Amap layers are the most reliable.

**Search can be slow.** Photon's servers are outside China; the first query may take 1-2 seconds.

---

## Files

```
WorldMap/
  index.html        - full version (recommended)
  static.html       - zero dependency version
  open-map.bat      - launches the full version
  open-static.bat   - launches the static version
  README.md         - this file
```
