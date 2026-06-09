# UI deep-dive — `com.android.SurfaceFlinger`, line by line

*This is the exhaustive companion to [Chapter 7](07-the-ui-plugin.md). Chapter 7 is the narrative; this is the algorithms — every non-trivial function, the exact control flow, and the edge cases. If you're modifying the plugin, read this. Code at `ui/src/plugins/com.android.SurfaceFlinger/`.*

## Module graph

```
index.ts ─ registers the page (surfaceflinger_page) + per-display tracks (surfaceflinger_track)
   │         and owns the one SurfaceFlingerSession
   ├─ surfaceflinger_session.ts ── all state + async loading; uses surfaceflinger_data
   ├─ surfaceflinger_page.ts ────── toolbar + SfViewer
   │      └─ surfaceflinger_viewer.ts ── the 3 panes
   │            ├─ surfaceflinger_controls.ts ─ the shared Surface toolbar
   │            ├─ surfaceflinger_rects.ts ──── the 2D-canvas 3D view
   │            └─ surfaceflinger_curated.ts ── the curated property summary
   └─ surfaceflinger_track.ts ──── the SliceTrack + SfSnapshotPanel (reuses SfRectsView)
   data: surfaceflinger_data.ts (SQL), route: surfaceflinger_route.ts, styles: surfaceflinger.scss
```

Eleven files. Dependency-free except on Perfetto's public widgets and `base/assert`.

---

## 1. `surfaceflinger_data.ts`

The only SQL file. Exposes typed interfaces (`SfDisplay`, `SfLayer`, `SfSnapshot`, `SfTransform`, `SfArg`) and four queries + two helpers.

**`sqlInt(value)`** — the injection/sanity guard for the one interpolated value (a display id, which is an int64-as-string):
```ts
export function sqlInt(value: string): string {
  if (!/^-?\d+$/.test(value)) throw new Error(`Expected an integer literal, got: ${value}`);
  return value;
}
```

**`queryDisplays`** — `GROUP BY display_id`, `MAX(tr.group_id)` read as `NUM_NULL` so a rect-less display yields `group: undefined` (not aliased to real group 0). `display_id` read as `LONG`, stored `.toString()` (int64-safe).

**`querySnapshots(displayId)`** — `SELECT id, ts ... WHERE display_id = ${sqlInt(displayId)} ORDER BY ts`; `id` as `NUM`, `ts` as `LONG`.

**`queryLayers(snapshotId)`** — the wide LEFT-JOIN (Chapter 5.7). Per-row mapping notes:
- `layerId`, `parent`, `zrel`, `hwc` read as `LONG` then `Number(...)` (they're small).
- `argSetId`, `base64`, `depth`, `opacity`, `grp`, `x/y/w/h`, the six matrix floats are `NUM_NULL` → `?? undefined`.
- `groupId: it.grp ?? undefined` — the display-scoping key; undefined for rect-less layers.
- `rect`: only built when `w !== null && h !== null` (`hasRect`), so rect-less layers have `rect === undefined` and never draw.
- the transform always materializes (defaults to identity `dsdx:1, dsdy:1`).

**`queryArgs(argSetId)`** — reads `key, flat_key, int_value (LONG_NULL), string_value (STR_NULL), real_value (NUM_NULL), value_type (STR)`. The value/type resolution is explicit so the linter's strict-boolean rule is satisfied and `0`/null are unambiguous:
```ts
if (it.vt === 'string' && it.s !== null) { value = it.s; type = 'string'; }
else if (it.vt === 'bool')   { value = it.i !== null && it.i !== 0n ? 'true' : 'false'; type = 'bool'; }
else if (it.vt === 'real' && it.r !== null) { value = it.r; type = 'real'; }
else if (it.i !== null) { value = it.i; type = 'int'; }
else if (it.s !== null) { value = it.s; type = 'string'; }
```

**`fmtNum(n)`** — integers verbatim, reals to 3 dp stripped of trailing zeros, non-finite verbatim.

---

## 2. `surfaceflinger_session.ts` — the engine

### Fields

Display/snapshot/index; `layers` + `byRowId`/`byLayerId` maps; `selectedRowId`/`selectedArgs`; `keepLayerId` (persist selection across snapshots); diff state (`prevLayers`, `prevByLayerId`, `prevProtoByLayerId`, `prevArgByKey`); `hiddenLayerIds`/`pinnedLayerIds` (Sets keyed by layerId); `relZParentIds`; `loadingLayers`; the two tokens `loadToken`/`argToken`; and the shared `options`.

### `init()`
```ts
this.displays = await queryDisplays(this.trace.engine);
if (this.displays.length > 0) {
  const def = this.displays.find((d) => !d.isVirtual) ?? this.displays[0];   // prefer a real display
  await this.setDisplay(def.displayId);
}
```

### The load funnel (the race-safe core)

All three navigation entry points allocate a token and call `loadIndex`:
```ts
async setDisplay(displayId) {
  const token = ++this.loadToken;
  this.displayId = displayId;
  const snapshots = await querySnapshots(this.trace.engine, displayId);
  if (token !== this.loadToken) return;          // superseded between query and use
  this.snapshots = snapshots;
  await this.loadIndex(0, token);                 // reuse the same token (one logical op)
}
async setIndex(i)        { await this.loadIndex(i, ++this.loadToken); }
async setNearestTs(ts)   { /* find nearest by |ts - s.ts| */ await this.loadIndex(best, ++this.loadToken); }
```
`loadIndex(i, token)` re-checks the token after **every** await and uses `try/finally` for the loading flag:
```ts
private async loadIndex(i, token) {
  if (token !== this.loadToken) return;
  if (this.snapshots.length === 0) { this.index = 0; this.layers = []; this.loadingLayers = false; m.redraw(); return; }
  this.index = clamp(i);
  this.loadingLayers = true;
  try {
    const layers = await queryLayers(this.trace.engine, snap.snapshotId);
    if (token !== this.loadToken) return;
    this.layers = layers; this.byRowId = ...; this.byLayerId = ...;
    this.relZParentIds = new Set(layers.filter(l => l.zOrderRelativeOf > 0).map(l => l.zOrderRelativeOf));
    await this.loadPrev();
    if (token !== this.loadToken) return;
    // pick default selection: keep the same layerId if present, else front-most visible w/ rect
    let def; for (const l of layers) { if (this.keepLayerId !== undefined && l.layerId === this.keepLayerId) {def=l;break;}
                                       if (l.isVisible && l.rect && (def===undefined || (l.drawDepth??0) > (def.drawDepth??0))) def=l; }
    def ??= layers.at(0);
    this.selectedRowId = undefined; this.selectedArgs = [];
    if (def !== undefined) await this.selectLayer(def.rowId);
  } finally {
    if (token === this.loadToken) this.loadingLayers = false;   // only the winner clears it
  }
  m.redraw();
}
```
Why two tokens: `selectLayer` (below) uses a *separate* `argToken`, so a user clicking a layer mid-scrub doesn't bump `loadToken` and abort the in-flight `queryLayers`. (See Chapter 8.2 for the exact races.)

### `selectLayer(rowId)`
```ts
async selectLayer(rowId) {
  const token = ++this.argToken;
  this.selectedRowId = rowId;                 // synchronous (highlight updates immediately)
  const layer = this.byRowId.get(rowId);
  this.keepLayerId = layer?.layerId;
  const args = layer?.argSetId !== undefined ? await queryArgs(this.trace.engine, layer.argSetId) : [];
  if (token !== this.argToken) return;        // superseded by a newer selection
  this.selectedArgs = args;
  const prevArgByKey = new Map<string,string>();
  if (this.options.diff && layer) {
    const pl = this.prevByLayerId.get(layer.layerId);
    if (pl?.argSetId !== undefined) { const pa = await queryArgs(...); if (token !== this.argToken) return;
                                      for (const a of pa) prevArgByKey.set(a.key, String(a.value)); }
  }
  this.prevArgByKey = prevArgByKey;
  m.redraw();
}
```

### Diff
- `loadPrev()` (called inside `loadIndex`): if `!diff || index<=0`, clears prev state; else loads the previous snapshot's layers into `prevLayers`/`prevByLayerId` and a `prevProtoByLayerId` map of layerId→base64ProtoId.
- `diffStatusOf(layer)`: `!diff || index<=0` → 'unchanged'; not in prev → 'added'; `prevId === base64ProtoId` → 'unchanged'; else 'modified'. Exact, because identical protos share one interned id.
- `deletedLayers()`: layers present in prev but not in current (the hierarchy's ghost rows).
- `setDiff(on)`: sets the flag, reloads prev, re-selects (to populate `prevArgByKey`), redraws.

### Small accessors
`displayName`, `selectedDisplayGroup` (the selected display's `group`), `displayLayers()` (filter `layers` by that group, or all if group undefined), `currentSnapshot`, `selectedLayer`, `nameOf(layerId)`, `selectLayerByLayerId(id)`, `toggleHidden`/`togglePinned` (flip a Set + redraw).

---

## 3. `surfaceflinger_rects.ts` — the canvas

### `view()`
Renders a single `canvas.pf-sf-rects__canvas` with `onclick → onClick`, `oncreate` capturing the element via `assertIsInstance(v.dom, HTMLCanvasElement)` and drawing, and `onupdate → draw` (so every redraw repaints the same canvas — no remount, no flicker).

### `draw()` — the pipeline
1. Size the canvas to CSS × DPR; `ctx = assertExists(canvas.getContext('2d'))`; `setTransform(dpr,...)`, clear; `col = readColors(canvas)`.
2. **bounded** = layers that have a quad (rect with w,h>0) and aren't per-rect-hidden. **items** = bounded filtered by `showOnlyVisible`, sorted back-to-front by drawDepth.
3. Empty state: if `items.length===0`, draw the message — distinguishing "N hidden by only-visible" from "no bounds" (Chapter 7.3) — clear `hit`/`labelHit`, return.
4. Reserve the gutter: `gutterW = min(220, max(132, floor(cssW*0.32)))`; `sceneW = cssW - gutterW`.
5. Compute camera: `pitch = rotation·π/8`, `yaw = 1.5·pitch`; precompute sin/cos; `zStep = explode·160`.
6. Centre the scene (centroid of all corners, mid-Z) so rotation pivots about the middle.
7. `project(x,y,z)` — translate to centre, yaw about Y, pitch about X, return `{x,y,d}` (d = projected depth).
8. Project every layer's quad at its stacked Z; fit all projected points into `[margin, sceneW-margin] × [margin, cssH-margin]` (uniform scale + centring offsets); `toScreen(p)`.
9. Painter sort by projected depth (far first).
10. Parse the three fill colors to RGB once (`parseRgb`).
11. Build `drawn` (each layer's screen quad, order, selected, pinned) + record `hit` (rowId + polygon + order) for click testing.
12. **Fill + border** per layer: gradient → solid `rgb` darkened by `0.55 + 0.45·(order/(n-1))`; opacity → `rgba` with the layer's alpha; wireframe → no fill. Border width/color: pinned (gold, 3px) > selected (accent, 2.5px) > normal (border, 1px).
13. `drawLabels(...)`.

### `drawLabels` — the no-overlap algorithm
```
this.labelHit = [];
labelAll = drawn.length <= 30
cands = drawn.filter(labelAll ? (selected||pinned||visible) : (selected||pinned))
       .sort(selected/pinned first, then front-most)
       .slice(0, floor((cssH - 2·pad)/lineH))            // only as many as fit
anchorOf(quad) = the rightmost corner (closest to the gutter → short leader lines)
placed = cands.map(anchor).sort(by anchor y)
nextMin = pad
for i, it in placed:
    remainingAfter = placed.length-1-i
    maxY = cssH - pad - 3 - remainingAfter·lineH         // reserve room for labels below + descender
    it.y = clamp(max(it.ay, nextMin), ≤ maxY)
    nextMin = it.y + lineH                                 // guarantees ≥ lineH spacing
for it in placed:
    record labelHit {rowId, x0=sceneW, y0=it.y-lineH+3, y1=it.y+4}   // clickable gutter band
    draw leader line anchor→(labelX,it.y), anchor dot, then halo-stroked + filled text
    text = fitText(name, gutterW-18)                       // ellipsis-truncate to fit
```
This is what guarantees labels never overlap each other, never overlap the rects (they're in the gutter), and never clip the canvas bottom.

### `onClick(e)`
Translate to canvas coords; first test `labelHit` bands (a click in the gutter selects that label's layer); else point-in-polygon over `hit`, topmost (highest order) wins; call `onSelect(rowId)`. No explicit `m.redraw()` — Mithril redraws after the handler and `selectLayer` redraws again on arg load.

### Helpers
- `apply(t,x,y)` — the affine (Chapter 7.3).
- `quad(layer)` — 4 transformed corners, or `undefined` if no rect / zero size.
- `pointInPoly(p,q)` — standard ray-cast.
- `readColors(el)` — `getComputedStyle` → the eight theme colors, with rgb fallbacks.
- `parseRgb(ctx,color)` — assign to `ctx.fillStyle` (normalizes to `#rrggbb`/`rgba`), parse to `{r,g,b}`; fallback grey on the rare unparseable form.
- `fitText(ctx,name,maxW)` — binary-ish trim with ellipsis until it fits.

---

## 4. `surfaceflinger_controls.ts`
`renderSurfaceControls(o)` returns the `.pf-sf-toolbar` with: a shading `Select` (gradient/opacity/wireframe, **first**), an "Only visible" `Checkbox`, and "Spacing"/"Rotation" range inputs (each maps 0–100 ↔ 0–1 on `o.explode`/`o.rotation`). `rectsOptionsFrom(o)` maps `{rectsOnlyVisible, explode, rotation, shading}` → the rects view's `SfRectsOptions`. Both the page and (the preview's options) come through here, so there's one source of truth.

---

## 5. `surfaceflinger_curated.ts`
`buildArgMaps(args)` → `{byKey, byFlat}` (by full key, and by flat key for repeated/array fields). Helpers: `s(byKey,key)` (formatted scalar), `rect(p)` → `(l, t) – (r, b)`, `color(p)` → `r:.. g:.. b:.. α:..`, `corners(arrayPrefix, scalarKey)` → `(tl, tr, bl, br)` with a scalar fallback, `px(key)` → `"<v> px"`, `transform(p)` → two matrix rows, `push(rows,label,value)` (skips empty). `curatedProperties(layer, args, layerName)` builds the sections in order: **Visibility** (visible; invisible-due-to from `visibility_reason[*]`; `occluded_by`/`partially_occluded_by`/`covered_by` as `layerRefs` for clickability; flags), **Geometry** (Z, layer stack, Z-relative-to, bounds, screen bounds, crop, destination frame, transform, requested transform), **Buffer** (active buffer w×h/stride/format, buffer transform, dataspace, frame number, HWC composition), **Color & effects** (color, requested color, corner radii, corner-radius crop, shadow/background-blur in `px`), **Input** (focusable, touchable region, input transform, input config, crop layer, replace-touch-with-crop). Empty sections are dropped.

---

## 6. `surfaceflinger_viewer.ts`
- `chipsFor(layer, dup, relZParent)` — the chip list (Chapter 7.6).
- `isDefaultish(a)` — bool 'false' / empty string / numeric 0 → default (a literal `"0"` string is **not** default).
- `simplify(name, on)` — if on and >3 dotted parts, `a.b.(...).z`.
- `buildTree(s)` — pool = layers; filter by only-visible; if a search term, keep matched layers **and their ancestors** (walk `parent` up); then if flat, return sorted by drawDepth; else group children under parents (`childrenOf` map; roots = layers whose parent isn't in the pool) and recurse `build(l)` sorted by drawDepth; append `deletedLayers()` as `deleted:true` ghosts when diff is on.
- `renderNode(s, item, dupIds)` — the clickable `.pf-sf-hnode` with id, simplified name, chips, and hide/pin `Button`s (with `e.stopPropagation()` so they don't also select); diff/selection CSS classes; recurses into a `TreeNode`.
- `renderHierarchy(s)` — computes `dupIds` (layerIds appearing more than once), the controls (Only visible/Flat/Simplify/Diff checkboxes + the filter `TextInput`), and the `Tree`.
- `renderCurated` / `buildCurated` — memoized by `(selectedLayer, selectedArgs)`; builds the rows (section header rows + value rows), the `headerTitles` set and `refByLabel` map, and the schema with the two cell renderers (header span; ref link); height = `min(count·27 + 40, 460)`.
- `renderProtoGrid` / `buildProto` — memoized by `(selectedArgs, propShowDefaults, diff, prevArgByKey)`; filters args by `propShowDefaults || !isDefaultish`, maps to `{property, value, type(, previous)}`.
- `renderPropHeader` / `renderProperties` / `renderSurface` — compose the panes (Chapter 7.3–7.6); the Surface uses `s.displayLayers()` (scoped) and `rectsOptionsFrom(o)`.

---

## 7. `surfaceflinger_track.ts`
`createSurfaceFlingerTrack(trace, uri, displayId, session)` — a `SliceTrack` whose dataset is this display's snapshots, `name = 'Frame ' || ROW_NUMBER() OVER (ORDER BY ts)`, with `colorizer: (row) => materialColorScheme(row.name)` and `detailsPanel: () => panel`. `SfSnapshotPanel`:
- `load(sel)` — if needed `setDisplay(displayId)`, then `setNearestTs(sel.ts)`; capture `displayName`, `index`/`total`, the display-scoped `layers`, `nLayers`/`nVisible`, and the top-6 visible-with-rect layer names.
- `render()` — `DetailsShell` (title + "Open in SurfaceFlinger viewer") wrapping a `GridLayout` of a "Snapshot" `Section` (Snapshot `N/total`, Timestamp, Layers, Visible, Top layers `Tree`) and a "Layout" `Section` (the `SfRectsView` preview, or "No layers."), each bound to the shared session.

---

## 8. `surfaceflinger_page.ts`
`SurfaceFlingerPage.view` — early-out empty state if no displays; else the `.pf-sf-bar` toolbar (display `Select` with "(virtual)" tags; the **Timeline** jump `Button` → `navigate('#!/viewer')` + `selectTrackEvent('/surfaceflinger_track/'+displayId, snapshotId, {scrollToSelection:true})`; first/prev/next/last `Button`s; the scrub `input[type=range]`; the `N / total` position handling `snapshots.length===0`; the `Timestamp`) over `.pf-sf-main` which is the `SfViewer` or the "No snapshots for this display." empty state.

---

## 9. `index.ts`
The plugin class (`id`, `description`, `onTraceLoad`): build the session, early-return if no displays, register the page + sidebar item, then the nested track group with one per-display track (Chapter 7.7). Plus the one line in `default_plugins.ts`.

---

## 10. `surfaceflinger.scss`
Flex 3-pane layout; panes `overflow:auto` + tree `min-width:max-content` (horizontal scroll, no clipping); all colors `var(--pf-color-*)`; diff tints + selection via `color-mix(... var(--pf-color-*) X%, transparent)`; the canvas background a checkerboard from theme vars; header card + curated-header + chip styles.

---

That's every file and every non-trivial algorithm. For the *why* behind the token logic, memoization, and the empty-state semantics, see the [Reviewer's guide](08-reviewers-guide.md).

[« Chapter 7](07-the-ui-plugin.md) · [README](README.md)
