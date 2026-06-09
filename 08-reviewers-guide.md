# Chapter 8 — Reviewer's guide

*Prerequisite: [Chapter 7](07-the-ui-plugin.md). This chapter is for someone reviewing or modifying the change: the invariants that must hold, the concurrency hazards and how they're handled, the design decisions worth knowing, the honest limitations, and a probe list.*

---

## 8.1 The change, in one paragraph

`com.android.SurfaceFlinger` is a Perfetto UI plugin that renders the `android.surfaceflinger.layers` data (Chapters 4–5) as a Winscope-style viewer: a per-display timeline track group of layer snapshots, and a full-screen page with three panes (Surface 3D rects, layer Hierarchy, Properties). It is delivered as **one commit on latest `upstream/main`**. Besides the UI, the commit carries a small trace_processor change in the **winscope importer** so a layers-only trace reports non-empty `trace_bounds` and actually opens (Chapter 5.6), plus the `ts cpp_access=READ` additions on the winscope snapshot tables in `winscope_tables.py` that the importer change requires (seven tables — the two that already had read access, shell-transitions and ProtoLog, are unchanged).

Commit footprint (no `storage_tables.cc`, no `build.mjs`):
- `src/trace_processor/plugins/winscope_importer/winscope_importer.cc` — bounds overrides
- `src/trace_processor/tables/winscope_tables.py` — `ts` read-access
- `ui/src/core/embedder/default_plugins.ts` — register the plugin (one line)
- `ui/src/plugins/com.android.SurfaceFlinger/` — the 11 plugin files

---

## 8.2 Concurrency: the invariants

The plugin loads everything async from the WASM engine while the user scrubs/clicks. The invariants:

1. **A superseded load writes nothing.** Every `await` in `loadIndex`/`selectLayer` is followed by a token re-check; if the token moved, the function returns before mutating session state. Without this, a slow query for snapshot A could land *after* a fast query for snapshot B and clobber B's layers (the user scrubs A→B, sees B, then the screen silently reverts to A). *Probe:* scrub rapidly across a long trace; the surface/hierarchy must always end on the snapshot the scrubber points at.

2. **Selecting a layer never cancels a layer load.** Load and selection use **separate** tokens (`loadToken`, `argToken`). A shared token would be wrong: clicking a layer mid-scrub would bump the token and abort the in-flight `queryLayers`, so the new snapshot's layers would never load. *Probe:* click layers while scrubbing; layers must keep loading.

3. **`loadingLayers` cannot get stuck.** It's cleared in a `finally`, guarded by `token === this.loadToken` so only the *winning* load clears it. *Probe:* rapid display switches; the view must never be stuck on a stale "Loading".

4. **Selection state is consistent.** `selectLayer` sets `selectedRowId` synchronously but only writes `selectedArgs` after re-checking `argToken`, so a selection landing during another can't show layer A's name with layer B's properties.

---

## 8.3 Performance: the invariants

- **DataGrid sources are memoized** by input identity (`selectedArgs` reference + `propShowDefaults` + `diff` + `prevArgByKey` reference). A fresh `InMemoryDataSource` per redraw would discard the grid's internal filter/sort cache on every hover/scrub tick. *Probe:* the memo key must include *every* input that changes the rows; if you add a row-affecting option, add it to the key, or the grid goes stale.
- **The canvas stays mounted across snapshot changes.** The rects view is rendered unconditionally (not swapped for a "Loading" div), so scrubbing redraws the same canvas element rather than remounting it — no flicker. The previous frame stays until the new layers arrive.
- **No per-frame surprises in `draw()`:** colors are read once per draw via `getComputedStyle` (cheap), the hit/label-hit arrays are rebuilt each draw (bounded by layer count), and the canvas is resized to the DPR each draw (a no-op when unchanged).

---

## 8.4 Design decisions a reviewer will ask about

- **Why is the bounds fix in the importer, not the UI or `storage_tables`?** The UI physically can't set `trace_bounds` (computed before the UI runs); `storage_tables` is the wrong, shared owner; the importer owns the tables it populates, exactly like the video/audio frame importers. (Chapter 5.6.)
- **Why per-display tracks, not one global track?** A snapshot is global, but each display composites a distinct layer set (Chapter 3.4); the encoder/recorder virtual display has genuinely different content. The earlier "single track / hide virtual" idea was wrong: it hid real data. Per-display tracks scoped by `group_id` show each display honestly. The apparent "duplicate" of two same-timed slices is resolved by the previews differing per display.
- **Why keep virtual displays?** They're real SF displays (Chapter 6.3). Hiding them would drop data the user may want (what the encoder composited). They're marked "(virtual)"; the default selected display is a non-virtual one.
- **Why derive header rows / refs from visible columns in the curated DataGrid?** `InMemoryDataSource` projects rows to the declared columns, so hidden marker fields don't survive; the cell renderers reconstruct header-ness (`headerTitles`) and refs (`refByLabel`) from the visible `property`/`value`.
- **Why a pure-2D canvas instead of WebGL/Three.js?** It renders headless (CI, the web UI) where the standalone Winscope's WebGL view is blank, and avoids a heavy dependency. The 3D is an orthographic projection of stacked quads.
- **`base64_proto_id` for diff:** identical protos share one interned id, so id-equality is an exact change test — cheaper and more correct than field-by-field comparison.

---

## 8.5 Known limitations (honest)

Things the standalone Winscope does that this plugin does not yet, mostly because a layers-only trace doesn't carry the data:

- **Rects zoom/pan** and rounded-corner rendering are not implemented (rotation/spacing/shading/hide/pin/select/labels are).
- **Color-by-content, touch-pointer dots, input rays, fill-regions** need ViewCapture / input-event data that a pure SF-layers trace lacks.
- **Relative-Z tree reparenting** in the hierarchy: the RelZ/RelZParent chips convey the relationship, but the tree isn't physically reparented to the relative parent.
- **Diff move-detection** (added-moved / deleted-moved) isn't distinguished beyond added/modified/deleted.
- **Scope is SurfaceFlinger only.** WindowManager / Transactions / IME / Transitions / ProtoLog are separate Winscope viewers (the importer already populates their tables; the bounds fix already covers them — adding their UIs is future work).

These are catalogued so a reviewer isn't surprised and a user knows what to expect.

---

## 8.6 Capture & test

**Minimal capture** (no ftrace needed, thanks to the bounds fix):
```
buffers { size_kb: 32768 }
duration_ms: 5000
data_sources { config {
  name: "android.surfaceflinger.layers"
  surfaceflinger_layers_config {
    mode: MODE_ACTIVE
    trace_flags: TRACE_FLAG_INPUT
    trace_flags: TRACE_FLAG_COMPOSITION
    trace_flags: TRACE_FLAG_VIRTUAL_DISPLAYS
  }
}}
```
```bash
adb shell 'cat cfg | perfetto --txt -c - -o /data/misc/perfetto-traces/sf.perfetto-trace'
adb pull /data/misc/perfetto-traces/sf.perfetto-trace
```

**Multi-display test:** run `adb shell screenrecord` during the capture to get a `ScreenRecorder` virtual display, then verify: the timeline shows a "SurfaceFlinger" group with a track per display; the page selector lists both (virtual marked); switching displays re-scopes the Surface (the composition changes); the virtual display shows the "No visible layers — N hidden" message until you uncheck "Only visible".

**Bounds fix:** load a layers-only trace and confirm it opens (pre-fix it showed empty). With the native shell: `trace_processor_shell -q <(echo 'SELECT start_ts, end_ts FROM trace_bounds;') sf.perfetto-trace` must be non-zero.

**Lint/typecheck (must be clean):**
```bash
node_modules/.bin/eslint src/plugins/com.android.SurfaceFlinger/
node_modules/.bin/tsc --noEmit -p tsconfig.json
```
The repo enforces `@typescript-eslint/strict-boolean-expressions`, `consistent-type-imports`, `curly`, and `no-irregular-whitespace` — all of which bit early drafts; keep them green.

---

## 8.7 A probe checklist

- [ ] Scrub fast across a long trace → view always lands on the right snapshot (token invariant 1).
- [ ] Click layers while scrubbing → layers still load (invariant 2).
- [ ] Switch displays rapidly → never stuck on "Loading" (invariant 3).
- [ ] Select a layer → name and properties always match (invariant 4).
- [ ] Toggle Diff → added/modified/deleted tints + Previous column; toggling off restores.
- [ ] Hide/pin a layer, scrub away and back → state persists (keyed by layerId).
- [ ] Rotate + spacing → labels fan out, never overlap, never clip the canvas; click a label → selects.
- [ ] Light/dark toggle → canvas and DOM both retheme.
- [ ] A display with no snapshots → "No snapshots for this display"; no displays → the record-this hint.
- [ ] Open-in-viewer ↔ jump-to-timeline → land on the same frame, same index/timestamp.

[« Chapter 7](07-the-ui-plugin.md)  ·  [Glossary »](glossary.md)
