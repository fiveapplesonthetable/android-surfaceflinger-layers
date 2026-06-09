# Chapter 7 — The UI plugin, annotated

*Prerequisite: [Chapter 5](05-trace-processor.md) (the tables) is essential; Chapters 1–3 give you the meaning of what's on screen. This chapter walks every file of `com.android.SurfaceFlinger`, teaching the Perfetto UI surface from zero as it goes.*

Code is at `ui/src/plugins/com.android.SurfaceFlinger/`. The plugin is ~1,400 lines across eleven files. We'll go in dependency order: framework background → wiring → data → session → the three panes → the timeline track → styles.

---

## 7.0 What you need before the code: Mithril and the plugin API

Everything below rests on two things: the Mithril UI framework and the Perfetto plugin API. Here's the minimum to read the code; the [glossary](glossary.md) defines every API name and widget this chapter mentions in passing.

**Mithril in six lines.** A component is an object with a `view()`. `view()` returns a tree of *vnodes* built by `m()`:

```ts
m('div.bar',                  // tag + CSS class — like writing <div class="bar">
  {onclick: () => doThing()}, // attrs: DOM attributes + event handlers
  ['hello', m('span', x)]);   // children: strings or more vnodes
```

There is no state store and no `setState`. Mithril re-runs `view()` and diffs the result against the real DOM after every event handler, and after an explicit `m.redraw()` (which is how *async* results — a finished SQL query — get on screen). So the rule of thumb when reading this plugin is: **the view is a pure function of the session object; anything async ends in `m.redraw()`.** Lifecycle hooks `oncreate`/`onupdate` run when a vnode's real DOM element (`v.dom`) is first attached and on each later redraw — §7.3 uses them to drive the `<canvas>`.

**The plugin API surface.** A plugin is a class implementing `PerfettoPlugin`; Perfetto calls its `onTraceLoad(ctx)` once per trace, and only if the plugin is listed in `default_plugins.ts`. The `ctx` argument (a `Trace`) is the entire API, and this plugin touches four corners of it:

| `ctx.` | what it does | used in |
| --- | --- | --- |
| `engine` | run SQL against the trace (async, WASM round-trip) | §7.1 |
| `pages` / `sidebar` | register the full-screen page + its menu item | §7.7 |
| `tracks` / `defaultWorkspace` | register timeline tracks, place them in lanes | §7.7 |
| `selection` / `navigate` | drive the app — select a slice, change route | §7.9 |

Two API objects do the heavy lifting in the body of the plugin. A **track** is one horizontal timeline lane: `SliceTrack.create(...)` builds one from a SQL query yielding `id, ts, dur, name, depth`, and tracks nest under `TrackNode`s. A **`DataGrid`** is Perfetto's sortable/filterable table — you hand it a schema, columns, and a `DataSource` of rows, and a column's `cellRenderer` can return any `m()` content. Both reappear with real code below.

The plugin uses only native Perfetto widgets — `Section`, `Tree`/`TreeNode`, `Chip`, `DataGrid`, `Checkbox`, `Select`, `TextInput`, `Button`, `Timestamp` — so it inherits the app's look and light/dark theming for free.

---

## 7.1 `surfaceflinger_data.ts` — the data layer

This is the only file that talks SQL. It defines the typed shapes (`SfDisplay`, `SfLayer`, `SfSnapshot`, `SfTransform`, `SfArg`) and the four queries.

`queryDisplays` lists displays with their group (layer stack) and virtual flag — exactly the Chapter 5 query:

```ts
SELECT
  d.display_id AS id,
  COALESCE(MAX(d.display_name), 'Display') AS name,
  MAX(IFNULL(d.is_virtual, 0)) AS virt,
  MAX(tr.group_id) AS grp
FROM __intrinsic_surfaceflinger_display d
LEFT JOIN __intrinsic_winscope_trace_rect tr ON tr.id = d.trace_rect_id
GROUP BY d.display_id
ORDER BY d.display_id
```

Two deliberate choices here, both from the reviewer's notes (Chapter 8):
- `display_id` is read as `LONG` and stored as a **string** (`it.id.toString()`) because a display id is an int64 that can exceed JavaScript's 2⁵³ safe integer.
- `grp` is read as `NUM_NULL` and stored as `number | undefined` — *not* `IFNULL(...,0)`. A display with no trace rect gets `group: undefined`, so it is **not** aliased onto the real group 0. (Group 0 is a legitimate display; conflating "no group" with "group 0" would scope the surface wrong.)

`querySnapshots(displayId)` returns the snapshots that include a display, in time order — and validates the interpolated id first:

```ts
WHERE display_id = ${sqlInt(displayId)}
```
```ts
// displayId is an int64 rendered as a base-10 string; validate before splicing
// into SQL so a malformed value fails loudly instead of a syntax error.
export function sqlInt(value: string): string {
  if (!/^-?\d+$/.test(value)) throw new Error(`Expected an integer literal, got: ${value}`);
  return value;
}
```

`queryLayers(snapshotId)` is the workhorse — it LEFT JOINs the layer to its decorated rect, the rect's geometry, and the transform matrix (the Chapter 5.7 query), and maps each row to an `SfLayer`. Note `groupId` comes from `tr.group_id` and is `undefined` for rect-less layers; `base64ProtoId` is the dedup id used for diffing; `hwcCompositionType` is the int from Chapter 2.5.

`queryArgs(argSetId)` reads the flattened proto fields from the generic `args` table — this is what the Properties pane's curated rows and proto dump are built from. The `value_type` column drives whether a value is read as int/real/string/bool (bool is `it.i !== null && it.i !== 0n` — handled explicitly so the linter's strict-boolean rule is satisfied and `0`/null are unambiguous).

`fmtNum` formats numbers compactly (integers as-is, reals to 3dp without trailing zeros).

---

## 7.2 `surfaceflinger_session.ts` — shared state, owned by the plugin

One `SurfaceFlingerSession` is created per trace and shared by **both** surfaces — the full-screen page and the timeline track's details panel. Because both surfaces read and write *one* object (selected display, snapshot list + index, loaded layers and lookup maps, selected layer + its args, diff state, hide/pin sets), they can't drift out of sync. The full field list is in the [deep-dive](ui-deep-dive.md).

### The view options (shared)

```ts
readonly options: SfViewOptions = {
  rectsOnlyVisible: true, explode: 0, rotation: 0, shading: 'gradient',
  hierOnlyVisible: false, hierFlat: false, hierSimplify: true, hierSearch: '',
  propShowDefaults: false, diff: false,
};
```
Both the page and the details-panel preview read these, so "rotate on the page" is reflected in the panel's inline preview, and vice-versa.

### Display scoping (the multi-display heart)

```ts
get selectedDisplayGroup(): number | undefined {
  return this.displays.find((d) => d.displayId === this.displayId)?.group;
}
// Layers composited on the selected display (by group). The surface view uses
// this so each display shows only its own composition; the hierarchy and
// properties use the full layer set (one cross-display tree).
displayLayers(): SfLayer[] {
  const g = this.selectedDisplayGroup;
  if (g === undefined) return this.layers;
  return this.layers.filter((l) => l.groupId === g);
}
```
This is the direct expression of Chapter 3.4: the **Surface** is scoped to the selected display's group; the **Hierarchy** stays the full tree. (A rect-less layer has `groupId === undefined` and is filtered out of the surface — correct, since it can't be drawn — but still appears in the hierarchy.)

### Async loading with two independent tokens (the race fix)

This is the most safety-critical code in the plugin. Loading is async (every `queryLayers`/`queryArgs` is a round-trip to the WASM engine), and the user can scrub or change displays faster than queries resolve. Two **monotonic tokens** discard superseded work:

```ts
private loadToken = 0;   // guards a display/snapshot/layer load
private argToken = 0;    // guards a selection's arg fetch
```
They are independent on purpose: **selecting a layer must never cancel an in-flight layer load.** `setDisplay` / `setIndex` / `setNearestTs` all funnel into `loadIndex(i, token)`, which re-checks the token after every `await` and only writes state if it's still current, with a `try/finally` so the loading flag can't get stuck:

```ts
private async loadIndex(i: number, token: number): Promise<void> {
  if (token !== this.loadToken) return;
  ...
  this.loadingLayers = true;
  try {
    const layers = await queryLayers(this.trace.engine, snap.snapshotId);
    if (token !== this.loadToken) return;            // superseded → bail, write nothing
    this.layers = layers; ...maps...
    await this.loadPrev();
    if (token !== this.loadToken) return;
    ...pick the default selected layer...
    if (def !== undefined) await this.selectLayer(def.rowId);
  } finally {
    if (token === this.loadToken) this.loadingLayers = false;
  }
  m.redraw();
}
```
`selectLayer` uses `argToken` the same way around its `queryArgs` calls, so a selection landing mid-scrub can't write stale args. (Chapter 8 explains the exact races these prevent.)

### Diff

`diffStatusOf(layer)` compares the layer's `base64ProtoId` to the same layer's id in the previous snapshot: not present → `added`; equal id → `unchanged`; different id → `modified`. Because identical protos share one interned id (Chapter 5.1), this is an **exact** "did anything change" diff, not a heuristic.

---

## 7.3 `surfaceflinger_rects.ts` — the Surface view (the 3D rects)

This is the signature view: a rotatable, stacked 3D rendering of the layer rectangles, drawn with a **pure 2D canvas** (no WebGL/Three.js, so it renders in headless CI where the standalone Winscope's WebGL view is blank).

### The projection, drawn

```
 layer bounds (x,y,w,h)
        │  apply affine transform:  x' = dsdx·x + dtdx·y + tx ,  y' = dtdy·x + dsdy·y + ty
        ▼
 a quad (4 corners)
        │  stack along Z by draw depth (zStep = explode · 160)
        ▼
 rotate: yaw around Y, then pitch around X      pitch = rotation · π/8 ;  yaw = 1.5 · pitch
        │  orthographic project (drop Z), fit into the canvas minus the right label gutter
        ▼
 screen quad ──► fill (gradient/opacity/wireframe) + border + leader-line label in the gutter
```

### The math, from the bottom up

A layer's rect is its bounds transformed by its affine matrix (Chapters 3.6, 4.4):
```ts
function apply(t: SfTransform, x: number, y: number): Pt {
  return {x: t.dsdx * x + t.dtdx * y + t.tx, y: t.dtdy * x + t.dsdy * y + t.ty};
}
```
Each layer becomes a quad (4 transformed corners). The quads are stacked along Z by draw depth and the whole scene is rotated and orthographically projected. The camera angles match SurfaceFlinger's own `mapper3d` so it feels like Winscope:
```ts
// pitch ramps to 22.5°, yaw is 1.5x pitch — both smooth from 0 so rotation=0 is flat.
const pitch = attrs.options.rotation * (Math.PI / 8);
const yaw = pitch * 1.5;
```
Spacing (the "explode" slider) sets the per-layer Z gap (`zStep = explode * 160`); the projection fits the result into the canvas minus a right-hand label gutter.

### Theme-aware colors on a canvas

A 2D canvas can't use CSS variables, so colors are read from the Perfetto theme at draw time via `getComputedStyle`, then round-tripped through `ctx.fillStyle` to RGB so gradient shading can darken by depth:
```ts
function readColors(el: HTMLElement): RectColors {
  const cs = getComputedStyle(el);
  const v = (name, fallback) => cs.getPropertyValue(name).trim() || fallback;
  return { visibleFill: v('--pf-color-success', ...), selectedFill: v('--pf-color-accent', ...),
           border: v('--pf-color-border', ...), label: v('--pf-color-text', ...),
           halo: v('--pf-color-background', ...), ... };
}
```
This is what makes the canvas follow light/dark mode along with the rest of the DOM. The three shading modes: **gradient** (solid fill darkened by depth, like Winscope), **opacity** (the layer's own alpha), **wireframe** (borders only).

### Leader-line labels (no occlusion — the part that took care)

Naïvely labelling each rect at its corner produces an unreadable pile when many full-screen layers share a corner. Instead labels live in a **right-hand gutter**, each in its own vertical slot, connected to its rect by a thin leader line + anchor dot — exactly the technique SurfaceFlinger's mapper3d uses. The placement is de-conflicted so labels never overlap and never clip the canvas bottom:
```ts
// each label reserves a slot, leaving room for the labels below it:
const maxY = cssH - pad - 3 - remainingAfter * lineH;   // -3 = descender headroom
let y = Math.max(placed[i].ay, nextMin);
if (y > maxY) y = maxY;
```
Labels are capped at 30 (like Winscope); above that, only the selected/pinned rects are labelled. The labels are **clickable**: their gutter rectangles are recorded in `labelHit` and checked in `onClick` before the rect hit-test, so clicking a label selects its layer.

### Click selection

The canvas records each drawn quad's screen polygon in `hit`; `onClick` does a point-in-polygon test (topmost wins) and calls `onSelect(rowId)`. Mithril redraws after the handler and `selectLayer` redraws again once args load, so there's no manual `m.redraw()` in the click path.

The canvas DOM and 2D context use the codebase's assertion helpers (`assertIsInstance(v.dom, HTMLCanvasElement)`, `assertExists(canvas.getContext('2d'))`) rather than casts — matching the VideoFrames plugin's conventions.

### Empty state, made actionable

If nothing draws, the message distinguishes "truly empty" from "everything is hidden by the only-visible filter" — the latter is what a `(virtual)` encoder display shows, since its only layer is an invisible mirror (Chapter 6.3):
```ts
const hiddenByVis = attrs.options.showOnlyVisible ? bounded.filter((d) => !d.l.isVisible).length : 0;
const msg = hiddenByVis > 0
  ? `No visible layers — ${hiddenByVis} hidden. Uncheck “Only visible” to show.`
  : 'No layers with bounds in this snapshot.';
```

---

## 7.4 `surfaceflinger_controls.ts` — the shared toolbar

A tiny module exporting `renderSurfaceControls(o: SfViewOptions)` (the shading `Select` + only-visible `Checkbox` + spacing/rotation sliders) and `rectsOptionsFrom(o)` (maps the session options to the rects view's options). Both the page's Surface pane and (historically) the details panel used it; it exists so there's one source of truth for the controls and no duplication. The shading dropdown is the first control (leftmost), then Only visible, then the sliders.

---

## 7.5 `surfaceflinger_curated.ts` — the "curated properties"

This derives Winscope's human-readable property summary for the selected layer from the flattened proto args, grouped into sections in a fixed order:

- **Visibility** — visible?, invisible-due-to reasons, occluded-by / partially-occluded-by / covered-by layer refs, flags.
- **Geometry** — Z, layer stack, Z-relative-to, bounds, screen bounds, crop, destination frame, transform, requested transform.
- **Buffer**, **Color & effects** (color, requested color, corner radii, requested corner radii, corner-radius crop, shadow radius, background blur — the last two in `px`), and **Input** (focusable, touchable region, input transform, input config, crop layer, replace-touch-with-crop).

Helpers format rects as `(l, t) – (r, b)`, colors as `r:… g:… b:… α:…`, and corner radii as `(tl, tr, bl, br)` (with a scalar fallback); layer-id references resolve to `name (#id)` so the UI can make them clickable.

---

## 7.6 `surfaceflinger_viewer.ts` — the 3-pane viewer

`SfViewer` lays out the three panes side by side and owns the Hierarchy and Properties rendering.

### Hierarchy

`buildTree` turns the flat layer list into the parent/child tree (filtered by only-visible and the search box, which keeps a matched layer's ancestors so the path stays visible), or a flat sorted list when "Flat" is on; with diff on it appends "ghost" rows for deleted layers. Each node renders its `layerId`, a (optionally simplified) name, and **chips**:
```ts
if (l.isVisible) chip('V', Success, 'Visible');
if (l.hwcCompositionType === 1) chip('GPU', Warning, 'Client/GPU composition');
else if (l.hwcCompositionType >= 2) chip('HWC', Primary, 'Hardware composer composition');
if (l.zOrderRelativeOf > 0) chip('RelZ', ...);
if (relZParent) chip('RelZParent', ...);
if (l.isHiddenByPolicy) chip('H', Danger, 'Hidden by policy');
if (l.isMissingZParent) chip('MissingZ', Danger, ...);
if (l.isSpy) chip('Spy', ...);
if (dup) chip('Dup', Warning, 'Duplicate layer id');
```
Every chip maps directly to a field from Chapters 3–5. Each node also has per-layer **hide** and **pin** toggle buttons (the hide/pin sets live on the session, keyed by `layerId`, so they persist as you scrub). "Simplify" collapses a long dotted class name `a.b.c.d.z` → `a.b.(...).z`.

### Properties — DataGrids, memoized

The Properties pane is a header card (selected layer name + chips) + the curated summary **as a DataGrid** + the full proto dump **as a DataGrid**. Because `InMemoryDataSource` projects rows to the declared columns (so you can't smuggle hidden fields), header-ness and clickable refs are derived from the two visible columns in the cell renderers:
```ts
property: { cellRenderer: (v) => headerTitles.has(String(v)) ? m('span.pf-sf-cur-h', String(v)) : String(v) },
value:    { cellRenderer: (v, row) => {
             const ref = refByLabel.get(String(row.property));
             return ref !== undefined ? m('a.pf-sf-ref', {onclick: () => s.selectLayerByLayerId(ref)}, String(v)) : String(v);
           } },
```
Both grids are **memoized** by input identity (`selectedArgs` reference + flags), so the many redraws from hovering/scrubbing don't reallocate the `DataGrid`'s data source and blow away its internal filter/sort cache:
```ts
const c = this.curatedCache?.layer === layer && this.curatedCache.args === s.selectedArgs
  ? this.curatedCache
  : (this.curatedCache = this.buildCurated(s, layer));
```
With diff on, the proto grid gains a **Previous** column showing the prior snapshot's value for any changed property.

---

## 7.6b `surfaceflinger_route.ts` — the one constant

The eleventh file is a one-liner, but it earns a mention because two surfaces must agree on it: it exports the page's route string.

```ts
export const SURFACEFLINGER_ROUTE = '/surfaceflinger';
```

`index.ts` registers the page at this route and the sidebar item links to `#!${SURFACEFLINGER_ROUTE}`; `surfaceflinger_page.ts`'s jump-to-timeline navigates relative to it. Keeping it in one place is why a future rename can't desync the page from its links.

## 7.7 `index.ts` — discovery & wiring

The plugin class:
```ts
export default class implements PerfettoPlugin {
  static readonly id = 'com.android.SurfaceFlinger';
  static readonly description =
    'SurfaceFlinger viewer: per-display layer snapshots on the timeline plus a ' +
    'full-screen rects/hierarchy/properties page. Inspired by the Android Winscope tool.';
  async onTraceLoad(ctx: Trace): Promise<void> { ... }
}
```
(That description string is the *one* place "Winscope" is named — everything else is `surfaceflinger`/`pf-sf-`, except the three `__intrinsic_winscope_*` SQL table names, which are upstream identifiers.)

`onTraceLoad` builds the session, returns early if there are no displays, then registers two surfaces:

- the **full-screen page** + a sidebar entry:
  ```ts
  ctx.pages.registerPage({ route: SURFACEFLINGER_ROUTE, render: (subpage) => m(SurfaceFlingerPage, {session, subpage}) });
  ctx.sidebar.addMenuItem({ section: 'current_trace', sortOrder: 35, text: 'SurfaceFlinger', href: `#!${SURFACEFLINGER_ROUTE}`, icon: 'layers' });
  ```
- a **nested "SurfaceFlinger" track group with one track per display** (real *and* virtual — virtual marked "(virtual)"):
  ```ts
  const group = new TrackNode({name: 'SurfaceFlinger', isSummary: true, sortOrder: -50});
  for (const display of session.displays) {
    const uri = `/surfaceflinger_track/${display.displayId}`;
    ctx.tracks.registerTrack({ uri, renderer: createSurfaceFlingerTrack(ctx, uri, display.displayId, session) });
    const name = display.isVirtual ? `${display.displayName} (virtual)` : display.displayName;
    group.addChildInOrder(new TrackNode({uri, name}));
  }
  ctx.defaultWorkspace.addChildInOrder(group);
  ```
This per-display grouping mirrors how the VideoFrames plugin groups streams, and it's correct because each display has a genuinely distinct composition (Chapter 3.4) — even if the snapshot *times* coincide.

---

## 7.8 `surfaceflinger_track.ts` — the timeline track + details panel

`createSurfaceFlingerTrack` makes a `SliceTrack` whose dataset is the snapshots that include this display, one incomplete slice (`dur = -1`) per snapshot — like the Screenshots/VideoFrames tracks — labelled with its **frame number** and colorized per frame:
```ts
SELECT id, ts, -1 AS dur, 0 AS depth, 'Frame ' || (ROW_NUMBER() OVER (ORDER BY ts)) AS name
FROM __intrinsic_surfaceflinger_layers_snapshot
WHERE id IN (SELECT snapshot_id FROM __intrinsic_surfaceflinger_display WHERE display_id = ${sqlInt(displayId)})
```
```ts
colorizer: (row) => materialColorScheme(row.name),
```

`SfSnapshotPanel` is the `TrackEventDetailsPanel` shown when you click a slice. Its `load()` syncs the shared session to *this* display + the nearest snapshot, then captures (for stability) the display name, snapshot index/total, layer counts, top layers, and the display-scoped layers for the preview. `render()` lays out a `GridLayout` of two sections. The left "Snapshot" section shows Snapshot `N / total`, Timestamp, Layers, Visible, and Top layers — its index and timestamp matching the page bar, so the two surfaces never disagree about which frame you're on. The right "Layout" section holds the inline rects preview plus an "Open in SurfaceFlinger viewer" button. The preview ships no controls of its own: it reuses the page's view options, which keeps the panel uncluttered and the two previews identical. Clicking a rect or label in it selects that layer, same as the page.

---

## 7.9 `surfaceflinger_page.ts` — the full-screen page

A toolbar over the 3-pane viewer. The toolbar has: a display **Select** (virtual displays shown as "Name (virtual)"), a **Timeline** jump button, first/prev/next/last snapshot buttons + a scrub slider, the `N / total` position, and a Timestamp. The jump button is the page→timeline bridge: it navigates to the timeline and **selects** the snapshot slice on this display's track (which opens its details panel and scrolls to it):
```ts
s.trace.navigate('#!/viewer');
s.trace.selection.selectTrackEvent(`/surfaceflinger_track/${s.displayId}`, cur.snapshotId, {scrollToSelection: true});
```
It handles the empty cases: no displays at all ("record with the android.surfaceflinger.layers data source"), and a selected display with no snapshots ("No snapshots for this display").

---

## 7.10 `surfaceflinger.scss` — theming

Every color is a Perfetto theme variable (`--pf-color-*`), so light/dark mode "just works" without per-mode code — the only reason it wouldn't is hardcoded colors, which there are none of (the canvas reads the same vars via JS, §7.3). The panes are flex with `overflow: auto` and `min-width: max-content` on the tree, so long names / deep trees / wide values scroll instead of clipping. The diff tints and selection use `color-mix(in srgb, var(--pf-color-...) X%, transparent)` so they adapt to the theme.

---

## 7.11 The two round-trips, end to end

- **Track → page:** click a snapshot slice → `SfSnapshotPanel.load()` syncs the session to that display+snapshot → "Open in SurfaceFlinger viewer" opens the page already on that frame.
- **Page → track:** "Timeline" button → `selectTrackEvent` selects the same snapshot slice on that display's track and scrolls to it.

Because both read one shared session and show the same snapshot index + timestamp, you move between the compact timeline panel and the full-screen page without losing which frame you're on.

[« Chapter 6](06-two-views-layers-vs-video.md)  ·  [Chapter 8 — Reviewer's guide »](08-reviewers-guide.md)
