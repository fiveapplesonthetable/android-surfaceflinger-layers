# Chapter 5 — From proto bytes to SQL tables

*Prerequisite: [Chapter 4](04-the-layers-trace.md). The trace is now a stream of `TracePacket`s, each carrying one `LayersSnapshotProto`. This chapter is how Perfetto's **trace_processor** flattens that nested proto into flat SQL rows the UI queries — the tables, the importer pipeline, rect/visibility computation, the two dedup tricks, and the one-commit **trace-bounds fix** that makes a layers-only trace open at all.*

Paths under `/mnt/agent/tmp/perfetto-vf` (branch `dev/zezeozue/winscope`).

---

## 5.1 The big picture

trace_processor is a C++ library that loads a trace and exposes SQL. It's also compiled to **WASM** (WebAssembly), so the exact same code runs inside the browser — that's how the Perfetto web UI queries a trace locally with no server, and why every query from the UI is an async round-trip (Chapter 7). For SurfaceFlinger layers there are three pieces:

1. **Intrinsic tables** — declared in `src/trace_processor/tables/winscope_tables.py`, from which C++ table classes (`tables::SurfaceFlingerLayerTable`, …) and SQL names (`__intrinsic_surfaceflinger_layer`, …) are generated.
2. **The importer** — `src/trace_processor/plugins/winscope_importer/`, a proto-importer module that decodes each packet and inserts rows.
3. **The trace-bounds contribution** — the importer reports its own time extent so a pure-layers trace isn't treated as empty (the fix).

Two cross-cutting tricks you'll see on every row:

- **`base64_proto_id`** — the raw proto bytes of a snapshot/layer, base64-encoded and **interned** into trace_processor's global **string pool**. *Interning* means storing each distinct value once in a shared table and handing back a small integer id; equal values get the same id, so identical blobs share one id. The UI fetches the original proto by this id. (It's a string-pool id, not a foreign key.) Because identical protos collapse to one id, **comparing two layers' `base64_proto_id` is an exact "did anything change?" test** — the plugin uses this for its diff (Chapter 7).
- **`arg_set_id`** — the whole proto is *also* flattened field-by-field into the generic `args` table. `arg_set_id` groups all args of one row, so the UI can show every proto field as key/value without a column per field.

---

## 5.2 The tables

From `src/trace_processor/tables/winscope_tables.py`. The SurfaceFlinger viewer touches six. Here's how one snapshot proto fans out into them:

```
LayersSnapshotProto (one TracePacket, field 93)
 ├─ snapshot ──► __intrinsic_surfaceflinger_layers_snapshot {ts, base64_proto_id, arg_set_id}
 ├─ DisplayProto[] ─► __intrinsic_surfaceflinger_display {snapshot_id, is_virtual, display_id,
 │                         trace_rect_id ─┐
 └─ LayerProto[]  ─► __intrinsic_surfaceflinger_layer {snapshot_id, layer_id, layer_name,
                          is_visible, hwc_composition_type, z_order_relative_of,
                          base64_proto_id, arg_set_id, layer_rect_id ─┐
                                                                      ▼  (both rect_ids point here)
                          __intrinsic_winscope_trace_rect {group_id (= layer_stack), depth,
                              is_visible, opacity,
                              rect_id ─────► __intrinsic_winscope_rect {x, y, w, h}        (deduped)
                              transform_id ► __intrinsic_winscope_transform {dsdx..ty}}    (deduped)

      every proto field ─► args {arg_set_id, key, value}      raw bytes ─► string pool (base64_proto_id)
```


### Geometry, deduplicated

```python
# __intrinsic_winscope_rect — one (x,y,w,h), shared
C('x', CppDouble()), C('y', CppDouble()), C('w', CppDouble()), C('h', CppDouble()),

# __intrinsic_winscope_transform — one 2x3 matrix, shared
C('dsdx', CppDouble()), C('dtdx', CppDouble()), C('tx', CppDouble()),
C('dtdy', CppDouble()), C('dsdy', CppDouble()), C('ty', CppDouble()),
```
Note the transform table **reunifies** the linear part (`dsdx…`) with the translation (`tx,ty`) that Chapter 4 said lived separately in the proto's `position`. The matrix maps `(x,y) → (dsdx*x + dtdx*y + tx, dtdy*x + dsdy*y + ty)`.

### The "decorated rect" — what layers and displays point at

```python
# __intrinsic_winscope_trace_rect
C('rect_id', CppTableId(WINSCOPE_RECT_TABLE)),     # FK -> the (x,y,w,h)
C('group_id', CppUint32()),                        # = layer_stack (the display grouping)
C('depth', CppUint32()),                           # sequential DRAW INDEX within the group
                                                   #   (a monotonic counter, NOT the layer's z;
                                                   #   the source var is confusingly named absolute_z)
C('is_spy', CppInt64()),
C('is_visible', CppInt64()),
C('opacity', CppOptional(CppDouble())),
C('transform_id', CppTableId(WINSCOPE_TRANSFORM_TABLE)),  # FK -> the matrix
C('border_width', CppOptional(CppDouble())),
C('border_color', CppOptional(CppUint32())),
```
This is the heart of the geometry model: a rect (the bounds) + a transform (the matrix) + per-frame attributes, with **`group_id` = the layer stack** (Chapter 3.4). The plugin's "which display does this layer belong to" is `group_id == display.group_id`.

### Snapshot, display, layer

```python
# __intrinsic_surfaceflinger_layers_snapshot  (one row per frame)
C('ts', CppInt64(), ColumnFlag.SORTED, cpp_access=CppAccess.READ),  # <- note cpp_access
C('arg_set_id', CppOptional(CppUint32()), ...),
C('base64_proto_id', CppOptional(CppUint32()), ...),

# __intrinsic_surfaceflinger_display  (one row per display per snapshot)
C('snapshot_id', CppTableId(SURFACE_FLINGER_LAYERS_SNAPSHOT_TABLE)),
C('is_on', CppInt64()), C('is_virtual', CppInt64()),
C('trace_rect_id', CppTableId(WINSCOPE_TRACE_RECT_TABLE)),  # the display's screen rect
C('display_id', CppInt64()), C('display_name', CppOptional(CppString())),

# __intrinsic_surfaceflinger_layer  (one row per layer per snapshot)
C('snapshot_id', CppTableId(SURFACE_FLINGER_LAYERS_SNAPSHOT_TABLE)),
C('arg_set_id', ...), C('base64_proto_id', ...),
C('layer_id', CppOptional(CppInt64())), C('layer_name', CppOptional(CppString())),
C('is_visible', CppInt64()),                 # COMPUTED visibility, not a raw field
C('parent', CppOptional(CppInt64())),
C('corner_radius_tl/tr/bl/br', CppOptional(CppDouble())),
C('hwc_composition_type', CppOptional(CppInt64())),
C('is_hidden_by_policy', CppInt64()),
C('z_order_relative_of', CppOptional(CppInt64())),
C('is_missing_z_parent', CppInt64()),
C('layer_rect_id', CppOptional(CppTableId(WINSCOPE_TRACE_RECT_TABLE))),  # the layer's bounds rect (nullable)
C('input_rect_id', CppOptional(CppTableId(WINSCOPE_TRACE_RECT_TABLE))),  # the input frame rect (nullable)
```

The `cpp_access=CppAccess.READ` on `ts` is required — without it the bounds fix wouldn't compile (§5.6).

---

## 5.3 The importer pipeline

The module registers for the snapshot packet field and dispatches it:

```cpp
// winscope_module.cc
case TracePacket::kSurfaceflingerLayersSnapshotFieldNumber:
    surfaceflinger_layers_parser_.Parse(timestamp, decoder.surfaceflinger_layers_snapshot(), sequence_id);
```

`Parse` (`surfaceflinger_layers_parser.cc:51`) orchestrates one snapshot → all rows:

```cpp
const auto& snapshot_id = ParseSnapshot(timestamp, blob, sequence_id);   // snapshot row
for (auto it = snapshot_decoder.displays(); it; ++it) ParseDisplay(...);  // display rows + group rects
const auto& layers_by_id = ExtractLayersById(layers_decoder);            // map id -> layer
const auto& layers_top_to_bottom = ExtractLayersTopToBottom(layers_decoder);  // DRAW ORDER
auto computed_visibility = VisibilityComputation(...).Compute();          // occlusion
const auto& computed_rects = RectComputation(...).Compute();              // screen rects
for (auto it = layers_decoder.layers(); it; ++it) ParseLayer(...);        // one row per layer
```

So the order is **snapshot → displays → flatten layers → compute visibility → compute rects → one layer row each.** Visibility and rects are computed for the whole tree up front because they need global z/occlusion context, then attached per layer.

### The draw order (`ExtractLayersTopToBottom`)

This recreates SurfaceFlinger's own in-order Z traversal (Chapter 3.3): sort each level by `(z, id)`, recurse `z < 0` children before the node and `z >= 0` children after, then reverse to get top-to-bottom (`surfaceflinger_layers_extractor.cc:43`, `:76`). The plugin doesn't re-implement this; it consumes `depth` (the per-group draw index the importer writes from this order).

---

## 5.4 Computing the screen rect (bounds × transform)

`RectComputation::InsertLayerTraceRectRow` (`surfaceflinger_layers_rect_computation.cc:325`) factors a layer's geometry into the two dedup tables:

```cpp
auto matrix = layer::GetTransformMatrix(layer_decoder);
auto bounds_rect = layer::GetBounds(layer_decoder);
row.rect_id = rect_tracker_.GetOrInsertRow(bounds_rect);     // the bounds (x,y,w,h)
row.group_id = layer_decoder.layer_stack();                  // group = layer stack
row.depth = static_cast<uint32_t>(absolute_z);
row.is_visible = is_computed_visible;
row.transform_id = transform_tracker_.GetOrInsertRow(matrix);  // the matrix
```

The bounds rect and the transform matrix are stored **separately** (so identical rects/matrices dedup), and the on-screen position is the bounds *transformed by* the matrix — applied by the UI when it draws (the plugin's `apply()`, Chapter 7) and by `TransformMatrix::TransformRect` (`winscope_geometry.cc:190`) for input frames. The transform itself is reconstructed from the proto's `type`+`position` or raw floats by `layer::GetTransformMatrix` (handling the orientation-flag case from Chapter 4.4). A display's rect comes from `MakeLayerStackSpaceRect` and its `group_id = display.layer_stack()` (`surfaceflinger_layers_parser.cc:325`).

The dedup trackers are plain get-or-insert caches keyed by value (`winscope_rect_tracker.cc:24`; equality uses an `EQUAL_THRESHOLD = 1e-6` defined in `winscope_geometry.cc:26`), so `rect_id`/`transform_id` are FKs into deduplicated geometry.

---

## 5.5 Computing visibility & occlusion

`VisibilityComputation::Compute` (`surfaceflinger_layers_visibility_computation.cc:206`) walks `layers_top_to_bottom` and, per layer:

```cpp
res.is_visible = IsLayerVisibleInIsolation(layer, excludes_composition_state);
if (res.is_visible) {
  for (const auto opaque_layer_id : opaque_layer_ids) {     // already-seen opaque layers above
    if (LayerContains(opaque_layer, layer, crop)) { res.is_visible = false; res.occluding_layers.push_back(...); }
    else if (LayerOverlaps(...))                            res.partially_occluding_layers.push_back(...);
  }
  for (translucent above) if (LayerOverlaps) res.covering_layers.push_back(...);
  if (IsOpaque(layer)) opaque_layer_ids.push_back(layer.id()); else translucent_layer_ids.push_back(...);
}
if (!res.is_visible) res.visibility_reasons = GetVisibilityReasons(...);
```

`IsLayerVisibleInIsolation` checks not-hidden-by-policy, alpha > 0, has content/effects, and a non-empty region (falling back to bounds when composition state is excluded). Because it walks **top to bottom** remembering opaque layers seen so far, a lower layer fully contained by an opaque higher one becomes invisible. The human-readable reasons ("flag is hidden", "buffer is empty", "alpha is 0", "occluded", …) are interned and assembled into the args.

The computed `is_visible` lands on both the layer row and its trace_rect; the occluded/partially-occluded/covered lists are added as args (`occluded_by`, `partially_occluded_by`, `covered_by`) — which the plugin renders as clickable layer references in the Visibility section (Chapter 7).

---

## 5.6 The trace-bounds fix (the change this book documents)

### The bug

A Perfetto trace has a `trace_bounds` (start_ts, end_ts) that the UI uses to set the visible timeline window. trace_processor computes it by asking each importer plugin for its time extent. The default plugin contributes **nothing**:

```cpp
// src/trace_processor/core/plugin/plugin.cc:151
std::pair<int64_t,int64_t> PluginBase::GetTimestampBounds() {
    return {std::numeric_limits<int64_t>::max(), 0};   // i.e. "no data"
}
```

If *every* plugin contributes nothing, the aggregate is `{0, 0}` and **the UI treats the trace as empty.** A trace captured with **only** `android.surfaceflinger.layers` has snapshots with real timestamps, but the winscope importer didn't report them — so `trace_bounds` was `[0,0]` and the layers viewer never even loaded. (This is the historical reason people had to add a dummy `linux.ftrace` data source to "make the trace open" — that padding is no longer needed.)

### The fix: the importer reports its own bounds

The winscope importer now overrides both bounds hooks, iterating the nine `ts`-bearing winscope snapshot tables. The shared rationale is the comment on `GetBoundsMutationCount` (`winscope_importer.cc:48`); `GetTimestampBounds` itself is just below it (`:63`):

```cpp
// src/trace_processor/plugins/winscope_importer/winscope_importer.cc:48 (comment), :63 (this method)
// The winscope snapshot tables are the only events in a trace captured with
// just the winscope data sources, so this plugin contributes their extent to
// the trace bounds (otherwise such a trace has empty bounds and the UI treats
// it as empty). GetBoundsMutationCount must enumerate the same tables.
std::pair<int64_t, int64_t> GetTimestampBounds() override {
  int64_t start_ns = std::numeric_limits<int64_t>::max();
  int64_t end_ns = 0;
  const auto& s = *trace_context_->storage;
  const auto include = [&](auto it) {
    for (; it; ++it) { start_ns = std::min(it.ts(), start_ns); end_ns = std::max(it.ts(), end_ns); }
  };
  include(s.surfaceflinger_layers_snapshot_table().IterateRows());
  include(s.surfaceflinger_transactions_table().IterateRows());
  include(s.windowmanager_table().IterateRows());
  include(s.window_manager_shell_transitions_table().IterateRows());
  include(s.inputmethod_clients_table().IterateRows());
  include(s.inputmethod_manager_service_table().IterateRows());
  include(s.inputmethod_service_table().IterateRows());
  include(s.viewcapture_table().IterateRows());
  include(s.protolog_table().IterateRows());
  return {start_ns, end_ns};
}
```

`GetBoundsMutationCount` enumerates **the same nine tables**' `.mutations()` — it's the cache key trace_processor uses to know whether to recompute bounds. The comment requires the two lists match; if `GetBoundsMutationCount` omitted the winscope tables, the mutation count wouldn't change when only winscope data loaded, and the recompute would be skipped — bounds would stay `[0,0]` even with the timestamp loop present. (This exact mismatch is a bug an early draft hit; the two methods must stay in sync.)

### Why a two-line table change rides along

`GetTimestampBounds` calls `it.ts()` from C++. By default a generated column is `CppAccess.NONE` — readable only from SQL, with **no C++ accessor**. So the same change flips every snapshot table's `ts` to C++-readable:

```diff
- C('ts', CppInt64(), ColumnFlag.SORTED),
+ C('ts', CppInt64(), ColumnFlag.SORTED, cpp_access=CppAccess.READ),
```

Without `cpp_access=CppAccess.READ`, `it.ts()` wouldn't exist and `winscope_importer.cc` wouldn't compile. So the importer change and the `winscope_tables.py` change are **one coupled fix**. (The bounds loop iterates *nine* winscope snapshot tables, but only *seven* needed this edit — two of them, shell-transitions and ProtoLog, already declared their `ts` C++-readable upstream.)

### Why this lives in the importer, not the UI plugin or the generic table plugin

This is a design question worth stating, because it's the kind of thing a reviewer asks:

- The **UI plugin cannot** set `trace_bounds` — those are computed by trace_processor *before* the UI runs; by the time `onTraceLoad` fires the timeline window is already set.
- It does **not** belong in the generic `storage_tables` plugin (which sweeps ftrace/sched/slice/etc. for bounds) — that's a different, shared plugin. Putting winscope tables there would be the wrong owner.
- It belongs in the **winscope importer**, exactly mirroring how the video and audio frame importers report the bounds of *their* tables. Each importer owns the bounds of the tables it populates. That's the idiomatic, consistent place.

### One honest wrinkle

A `transactions`-bearing capture carries old *buffered* transaction timestamps, so its bounds can span a huge range while the layer snapshots cover only a few seconds. A **SF-layers-only** capture is clean (bounds = the snapshot range). That's why the "simple capture" recommended in Chapter 4.5 is layers-only.

---

## 5.7 What you can now query

After import, the UI's entire data contract is these tables. A flavour (the real queries are in Chapter 7):

```sql
-- displays in the trace, with their group (layer stack) and virtual flag
SELECT d.display_id, MAX(d.display_name), MAX(d.is_virtual), MAX(tr.group_id)
FROM __intrinsic_surfaceflinger_display d
LEFT JOIN __intrinsic_winscope_trace_rect tr ON tr.id = d.trace_rect_id
GROUP BY d.display_id;

-- all snapshots (frames) in time order
SELECT id, ts FROM __intrinsic_surfaceflinger_layers_snapshot ORDER BY ts;

-- every layer of one snapshot, with its bounds rect + transform + group
SELECT l.layer_id, l.layer_name, l.is_visible, l.hwc_composition_type,
       tr.group_id, r.x, r.y, r.w, r.h, t.dsdx, t.dtdx, t.tx, t.dtdy, t.dsdy, t.ty
FROM __intrinsic_surfaceflinger_layer l
LEFT JOIN __intrinsic_winscope_trace_rect tr ON tr.id = l.layer_rect_id
LEFT JOIN __intrinsic_winscope_rect r        ON r.id = tr.rect_id
LEFT JOIN __intrinsic_winscope_transform t   ON t.id = tr.transform_id
WHERE l.snapshot_id = ?;
```

Next: a short but important detour — how the *other* display data source (video frames) relates to all this, and why a screen recording shows up as a virtual display here.

[« Chapter 4](04-the-layers-trace.md)  ·  [Chapter 6 — Two views of one display »](06-two-views-layers-vs-video.md)
