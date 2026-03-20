# VisAge

> Browser-native AoE2 DE replay visualizer — no install, no server, no build step.

Drop a `.aoe2record` file onto the page and watch unit paths, terrain, resources, age-up events, and live APM animate in real time.

<img width="780" height="654" alt="2026-03-20 14_01_54-AgeVis — AoE2 Replay Visualizer — Mozilla Firefox" src="https://github.com/user-attachments/assets/1c15d5a1-dfe5-4512-b090-2f43217f3592" />

---

## Features

- **Zero-dependency runtime** — single `index.html`, open locally in any WebGL-capable browser
- **Pure browser parsing** — Pyodide (WASM Python 3.12) + mgz runs entirely client-side
- **DE save 66.x and 67.x** — binary-patched against the save 67.20 struct changes that mgz 1.8.51 predates
- **Animated playback** — 1×/2×/4×/8×/16× speed, scrub timeline, Space/←/→ keyboard shortcuts
- **Live telemetry** — pop (alive/total), buildings, rolling 60s APM per player
- **Age-up markers** — Feudal/Castle/Imperial tick marks on timeline with player labels
- **Terrain bake** — offscreen canvas → WebGL sprite, elevation shading, gaia resources
- **Tooltips** — hover any entity for unit/building type and owner

---

## Usage

```
Open index.html in Firefox or Chrome
Drop a .aoe2record file onto the dropzone (or Browse File)
```

Supports **1v1 DE replays** (save versions 13.34–67.x tested). Multi-player replays will parse but only P1/P2 telemetry cards are shown.

---

## Architecture

```
.aoe2record
    │
    ▼
Pyodide 0.27.0  (WASM Python 3.12)
  └─ construct 2.10.67  ← tarball-installed, shimmed for Python 3.12 compat
  └─ aocref              ← reference data (player colors, civ names, speeds)
  └─ mgz 1.8.51          ← patched at runtime for save 67.x struct changes
    │  JSON payload
    ▼
JS Simulation
  └─ compilePaths()     — kinematic path array per entity, O(n) command walk
  └─ cullCombatDeaths() — O(n²) proximity heuristic, fine for 1v1
  └─ compileAPM()       — sorted timestamp arrays for O(log n) rolling window
    │  per-frame state
    ▼
PixiJS v8.9.2  (WebGL)
  └─ Pooled PIXI.Graphics (circles = units, squares = buildings)
  └─ Static terrain sprite (HTML5 canvas bake → texture)
```

### Why tarball-install instead of micropip?

Neither `construct` nor `mgz` ship WASM-compatible wheels. We fetch `.tar.gz` sources from the PyPI JSON API, extract to `sys.path`, then apply shims for the construct 2.8.x → 2.10.67 API gaps that mgz 1.8.51 was never updated for.

### Save 67.x patches

mgz 1.8.51's highest version gate is `66.6`. AoE2 DE patches released in 2025–2026 added:
- A new `field_A` uint32 per player slot in `parse_de` (breaks `custom_civ_count` loop)
- 8 trailing bytes at the end of `parse_de` before `parse_metadata`
- CRC fields in `string_block` grew from uint32 → uint64

All are patched at runtime via `inspect.getsource` + `exec` without forking mgz.

---

## Known Limitations

| Issue | Notes |
|---|---|
| 1v1 only | Multi-player sidebar hardcoded to 2 player cards |
| No zoom/pan | `PIXI.Container` has no viewport; Ludikris maps are tiny |
| Civ names wrong for P2 on save 67.x | `parse_player` object-section parser misaligned; names and paths correct |
| Training not simulated | Units appear only when they first receive a move command |
| No fog-of-war | Full map always visible |
| Mobile unsupported | Flex layout collapses; no touch scrub |

---

## Development

No build step. Edit `index.html` directly.

**Module layout** (all in one `<script>` tag):

| Object | Responsibility |
|---|---|
| `Config` | Magic numbers, color maps, speed cycle |
| `Logger` | Dual-channel log (sidebar DOM + console), counter badges |
| `Core` | Boot: PixiJS → data.json → Pyodide → Python deps |
| `Parser` | `PY_BOOT` / `PY_EXTRACT` strings + `processFile()` |
| `MathUtils` | `getPosAtTime()` — linear-scan lerp |
| `Simulation` | `compilePaths()`, `cullCombatDeaths()`, `compileAPM()` |
| `Render` | `buildTerrainCache()`, `drawFrame()` |
| `Tooltip` | DOM hover tooltip, singleton |
| `APMCalc` | O(log n) rolling 60s APM via binary search |
| `UI` | Event wiring, playback controls, keyboard shortcuts |

**Debugging:**

```javascript
// After a successful replay load:
Object.keys(AgeVisState.paths).length        // entity count
AgeVisState.paths[10045]                     // inspect one entity
AgeVisState.rawPayload.commands.filter(c => c.type === 'age_up')
AgeVisState.apm["1"].length                  // APM timestamp count P1
```

---

## Roadmap

- [ ] Zoom / pan (pixi-viewport or native Container transform)
- [ ] Unit training simulation (ActionEnum.CREATE / TRAIN → spawn entity)
- [ ] APM heatmap density overlay
- [ ] Econ overlay (resource totals from action payloads)
- [ ] Export PNG frame / GIF clip
- [ ] Multi-player sidebar extension (currently hardcoded 2 cards)
- [ ] Fix `parse_player` object-section alignment for save 67.x (civ names)
- [ ] Resize handling (canvas currently fixed 750 px)

---

## Dependencies

| Package | Version | Source |
|---|---|---|
| PixiJS | 8.9.2 | CDN |
| Pyodide | 0.27.0 | CDN (jsDelivr) |
| construct | 2.10.67 | PyPI tarball at runtime |
| aocref | latest | PyPI tarball at runtime |
| mgz | 1.8.51 | PyPI tarball at runtime |
| aoe2techtree data.json | master | SiegeEngineers GitHub |

---

## License

MIT
