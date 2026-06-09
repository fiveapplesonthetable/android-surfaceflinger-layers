# Chapter 4 — The layers trace

*Prerequisite: [Chapter 3](03-layers-and-displays-data-model.md). We now leave the device. This chapter is the exact bytes: how SurfaceFlinger serializes a snapshot of its layer tree, the proto schema field by field, the transform-matrix encoding (which trips everyone up), and the `android.surfaceflinger.layers` data source with its capture flags.*

Two source trees are involved:
- **The schema** lives twice — a legacy hand-written copy at `~/dev/aosp/development/tools/winscope/protos/surfaceflinger/udc/` and the live Perfetto copy at `~/dev/aosp/external/perfetto/protos/perfetto/trace/android/`. They're kept in sync by hand; the Perfetto copy is newer. Citations below use the Perfetto copy.
- **The producer** is SurfaceFlinger's `Tracing/` directory plus `LayerProtoHelper.cpp`.

---

## 4.0 Protobuf in 60 seconds (so the schema reads cleanly)

Every field in a `.proto` message has a name, a type, and a small integer **field number** (`= 1`, `= 2`, …). On the wire, protobuf stores only the *number* (as a tag), never the name. So:
- renaming a field is harmless; **reusing a number** for a different field corrupts old data;
- retired fields keep their number (`final_crop = 15 [deprecated=true]`) or are `reserved` (`reserved 1;`);
- proto2 `optional` gives true "has this field?" presence (used here), unlike proto3 scalars.

A **snapshot** is the entire layer tree (every layer) plus the displays, captured at one instant (one vsync). The trace is a time-ordered sequence of snapshots, one per `TracePacket`.

---

## 4.1 One snapshot: `LayersSnapshotProto`

```proto
// external/perfetto/protos/perfetto/trace/android/surfaceflinger_layers.proto:51
message LayersSnapshotProto {
  optional sfixed64 elapsed_realtime_nanos = 1;  // boot-relative time of the snapshot
  optional string   where = 2;                   // which SF stage triggered it
  optional LayersProto layers = 3;               // the whole layer tree (flat list)
  optional string   hwc_blob = 4;                // HWC state dump (only with TRACE_FLAG_HWC)
  optional bool     excludes_composition_state = 5;
  optional uint32   missed_entries = 6;
  repeated DisplayProto displays = 7;            // the displays in this snapshot
  optional int64    vsync_id = 8;
}
```

- **`elapsed_realtime_nanos`** — set from `time.ns()` (`SurfaceFlinger.cpp:9352`); this becomes the snapshot's `ts`.
- **`where`** — "Currently either `visibleRegionsDirty` or `bufferLatched`" (`:55`). It records *which commit phase* produced the snapshot.
- **`layers`** — a `LayersProto`, which is just `repeated LayerProto layers = 1` — a **flat** list; the tree is rebuilt from `id`/`parent`/`children`.
- **`excludes_composition_state`** — true when composition state (visible region, HWC type) was *not* captured (i.e. `TRACE_FLAG_COMPOSITION` was off). It's written as the negation of the flag (`SurfaceFlinger.cpp:9355`).
- **`displays`** — repeated `DisplayProto` (§4.4).
- **`vsync_id`** — used to de-dup generated snapshots.

The legacy on-disk file wraps these in a `LayersTraceFileProto { fixed64 magic_number = 1; repeated entry = 2; ... }` with a `.LYRTRACE` magic; but **through Perfetto each snapshot is a `TracePacket` field instead** (§4.6), so the file wrapper doesn't appear in Perfetto traces.

---

## 4.2 One layer: `LayerProto`

This is the big one (`surfaceflinger_layers.proto:116`). Grouped by what the plugin uses:

**Identity & tree**
- `id = 1` unique id; `name = 2` (e.g. `"StatusBar#75"`; SF appends `"(Mirror)"` for clones, `LayerProtoHelper.cpp:468`); `type = 5`; `parent = 25` (`-1` if none); `children = 3`; `relatives = 4`; `z_order_relative_of = 26`; `is_relative_of = 51`; `original_id = 58`.

**Geometry & placement**
- `layer_stack = 9` (uint32 — *this is the display grouping*); `z = 10`; `position = 11` / `requested_position = 12` (`PositionProto {x,y}` — the translation, since `TransformProto` omits it, see §4.5); `size = 13`; `crop = 14`; `bounds = 45`, `screen_bounds = 46`, `source_bounds = 44` (`FloatRectProto`); `destination_frame = 57`.
- `transform = 23`, `requested_transform = 24`, `buffer_transform = 39`, `effective_transform = 43` — all `TransformProto` (§4.5).
- `corner_radius = 41` (`[deprecated]` — the live one is the per-corner `corner_radii = 61`, a `CornerRadiiProto {tl,tr,bl,br}`), `corner_radius_crop = 48`, `shadow_radius = 49`, `background_blur_radius = 52`.

**Appearance & composition**
- `is_opaque = 16`; `color = 20` / `requested_color = 21` (`ColorProto {r,g,b,a}` normalized 0–1); `flags = 22` (bitfield: `hidden=0x01, opaque=0x02, secure=0x80`); `dataspace = 18` (e.g. `"STANDARD_BT709"`); `pixel_format = 19`.
- `hwc_composition_type = 35` — the `HwcCompositionType` enum (the same DEVICE/CLIENT/… from Chapter 2.5):
  ```proto
  // surfaceflinger_layers.proto:97
  HWC_TYPE_UNSPECIFIED = 0; HWC_TYPE_CLIENT = 1; HWC_TYPE_DEVICE = 2;
  HWC_TYPE_SOLID_COLOR = 3; HWC_TYPE_CURSOR = 4; HWC_TYPE_SIDEBAND = 5;
  HWC_TYPE_DISPLAY_DECORATION = 6;
  ```
- `active_buffer = 27` (`ActiveBufferProto {width,height,stride,format,usage}`, "null if there's nothing to draw"); `hwc_frame = 30`/`hwc_crop = 31`/`hwc_transform = 32`; `curr_frame = 37`.

**Regions & input**
- `transparent_region = 6`, `visible_region = 7`, `damage_region = 8` (each a `RegionProto` = list of `RectProto`).
- `input_window_info = 47` (`InputWindowInfoProto`) — only when `TRACE_FLAG_INPUT` (`LayerProtoHelper.cpp:502`). It carries `focusable`, `frame`, `touchable_region`, `transform`, `crop_layer_id`, `replace_touchable_region_with_crop`, `touchable_region_crop`, and the packed `input_config` (field 17).

Small geometry messages, for completeness: `RectProto {int32 left,top,right,bottom}`; `FloatRectProto` (same, float); `PositionProto {float x,y}`; `ColorProto {float r,g,b,a}`; `RegionProto { reserved 1; repeated RectProto rect = 2; }`.

---

## 4.3 `DisplayProto`

```proto
// surfaceflinger_layers.proto:83
message DisplayProto {
  optional uint64 id = 1;
  optional string name = 2;                      // "Built-in Screen"
  optional uint32 layer_stack = 3;               // which stack composites here
  optional SizeProto size = 4;
  optional RectProto layer_stack_space_rect = 5; // the logical region mapped to the display
  optional TransformProto transform = 6;
  optional bool is_virtual = 7;                  // physical vs virtual
  optional double dpi_x = 8;
  optional double dpi_y = 9;
}
```

`layer_stack` here is the join key against each layer's `layer_stack` (Chapter 3.4): it's what the importer turns into the display's `group_id`. `is_virtual` is what the plugin shows as "(virtual)".

---

## 4.4 The transform encoding (the part everyone gets wrong)

```proto
// surfaceflinger_common.proto:34
message TransformProto {
  optional float dsdx = 1;
  optional float dtdx = 2;
  optional float dsdy = 3;
  optional float dtdy = 4;
  optional int32 type = 5;
}
```

The four floats are the **2×2 linear part** of the affine matrix:

```
| dsdx  dsdy |     "s" = the x-like output coord, "t" = the y-like output coord.
| dtdx  dtdy |     dsdx = ∂(out x)/∂(in x), dtdx = ∂(out y)/∂(in x), etc.
```

**Translation `tx`/`ty` are NOT in `TransformProto`.** They're carried separately in the layer's `position` (`PositionProto`), filled from `transform.tx()/ty()` (`LayerProtoHelper.cpp:444`). So to reconstruct a layer's full matrix you combine `TransformProto` (linear) + `position` (translate). *(The plugin's `__intrinsic_winscope_transform` table re-unifies them into one 6-float `{dsdx,dtdx,tx,dtdy,dsdy,ty}` row — Chapter 5.)*

The `type` field is packed: low byte = a type mask, bits 8+ = the rotation/orientation flags:

```cpp
// LayerProtoHelper.cpp:153
const uint32_t type = transform.getType() | (transform.getOrientation() << 8);
```
- type mask (`Transform.h:55`): `IDENTITY=0, TRANSLATE=0x1, ROTATE=0x2, SCALE=0x4, UNKNOWN=0x8`;
- orientation (`Transform.h:45`): `ROT_0=0, FLIP_H=1, FLIP_V=2, ROT_90=4, ROT_180=3, ROT_270=7, ROT_INVALID=0x80`.

**Why the floats are sometimes 0 in the proto:** they're only serialized for `SCALE`/`UNKNOWN` transforms. For identity/flip/90/180/270, the reader can regenerate the 2×2 from the orientation flags, so the floats are left at their proto defaults to save bytes:

```cpp
// LayerProtoHelper.cpp:159
if (type & (ui::Transform::SCALE | ui::Transform::UNKNOWN)) {
    transformProto->set_dsdx(...); set_dtdx(...); set_dsdy(...); set_dtdy(...);
}
```

This is why the plugin's affine application (`x' = dsdx*x + dtdx*y + tx`, Chapter 7) and the importer's matrix reconstruction both have to handle the orientation-flag case, not just the raw floats.

---

## 4.5 The data source: `android.surfaceflinger.layers`

SurfaceFlinger registers a Perfetto data source named exactly that:

```cpp
// services/surfaceflinger/Tracing/LayerDataSource.h:64
    static constexpr auto* kName = "android.surfaceflinger.layers";
```

It bumps the shared-memory hint because each snapshot is large ("~50kb/entry and the default shmem buffer size (256kb) could be overrun", `LayerDataSource.cpp:29`). Its config message:

```proto
// external/perfetto/protos/perfetto/config/android/surfaceflinger_layers_config.proto:22
message SurfaceFlingerLayersConfig {
  enum Mode {
    MODE_UNSPECIFIED = 0;
    MODE_ACTIVE = 1;     // snapshot every time layers change
    MODE_GENERATED = 2;  // generate snapshots from the transaction ring buffer on flush
    MODE_DUMP = 3;       // a single snapshot
    MODE_GENERATED_BUGREPORT_ONLY = 4;  // default; like GENERATED but only for bugreports
  }
  optional Mode mode = 1;
  enum TraceFlag {
    TRACE_FLAG_INPUT = 0x02;
    TRACE_FLAG_COMPOSITION = 0x04;
    TRACE_FLAG_EXTRA = 0x08;
    TRACE_FLAG_HWC = 0x10;
    TRACE_FLAG_BUFFERS = 0x20;
    TRACE_FLAG_VIRTUAL_DISPLAYS = 0x40;
  }
  repeated TraceFlag trace_flags = 2;
}
```

### What each Mode means

- **`MODE_ACTIVE`** — the one the plugin's "simple capture" uses. A snapshot is taken every time the layer state changes (plus one immediately on start so the trace has an initial state, `LayerTracing.cpp:56`). Per vsync, `doActiveLayersTracingIfNeeded` decides whether to snapshot (`SurfaceFlinger.cpp:9329`).
- **`MODE_GENERATED`** — replays SF's internal transaction ring buffer at flush time to *reconstruct* snapshots after the fact (used for bugreports; the output is hundreds of MB).
- **`MODE_DUMP`** — a single snapshot at stop.
- **`MODE_GENERATED_BUGREPORT_ONLY`** — the default if `mode` is unspecified (`LayerDataSource.cpp:48`).

### What each trace flag adds (this maps directly to plugin features)

- **`TRACE_FLAG_INPUT (0x02)`** — adds `input_window_info` per layer (the plugin's Input curated section + Spy chip). Gated at `LayerProtoHelper.cpp:502`.
- **`TRACE_FLAG_COMPOSITION (0x04)`** — adds composition state: visible region and `hwc_composition_type` (the GPU/HWC chips). Also flips `excludes_composition_state` to false. In active mode it selects the post-composition commit phase to snapshot (`SurfaceFlinger.cpp:9335`).
- **`TRACE_FLAG_VIRTUAL_DISPLAYS (0x40)`** — includes layers belonging to **virtual displays** (the encoder/recorder display from Chapter 3.5). Without it, virtual-display layer stacks are skipped (`SurfaceFlinger.cpp:6691`). The plugin's recommended capture config sets this so you can inspect the recorder display.
- **`TRACE_FLAG_HWC (0x10)`** — fills the snapshot's `hwc_blob`.
- **`TRACE_FLAG_BUFFERS (0x20)`** — in active mode, also snapshots on `bufferLatched` (not only `visibleRegionsDirty`).
- **`TRACE_FLAG_EXTRA (0x08)`** — per-layer `metadata` map + the off-screen layer hierarchy.

The config rides on the generic `DataSourceConfig` as field 121 (`data_source_config.proto:213`).

### The minimal capture, explained

This is the config the plugin's demo uses (Chapter 7 / the project README). It's just the one data source — **no ftrace needed** (the trace-bounds fix in Chapter 5 is what makes a layers-only trace open):

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

`MODE_ACTIVE` → a snapshot per frame (so you can scrub/diff). The three flags fill in input regions, the HWC/GPU composition chips, and the virtual displays — i.e. exactly the columns the plugin renders. Drop the flags and those parts simply show empty; drop `MODE_ACTIVE` for `MODE_DUMP` and you get a single snapshot.

---

## 4.6 How a snapshot becomes a `TracePacket`

`LayerTracing::writeSnapshotToPerfetto` serializes the `LayersSnapshotProto` once and emits it as a trace packet, timestamped on `CLOCK_MONOTONIC`:

```cpp
// LayerTracing.cpp:214
    auto packet = context.NewTracePacket();
    packet->set_timestamp(snapshot.elapsed_realtime_nanos());
    packet->set_timestamp_clock_id(... BUILTIN_CLOCK_MONOTONIC);
    auto* snapshotProto = packet->set_surfaceflinger_layers_snapshot();
    snapshotProto->AppendRawProtoBytes(snapshotBytes.data(), snapshotBytes.size());
```

The packet field:
```proto
// external/perfetto/protos/perfetto/trace/trace_packet.proto:231
    LayersSnapshotProto surfaceflinger_layers_snapshot = 93;
```

So in a finished Perfetto trace, **each SF layers snapshot is one `TracePacket` whose field 93 is a `LayersSnapshotProto`.** Everything `set_*` on a layer is reachable from there. (To avoid deadlocks, SF schedules the snapshot on its main thread and stops asynchronously — `SurfaceFlinger.cpp:1027`, b/313130597.)

Next: how trace_processor turns these packets into the SQL tables the plugin queries.

[« Chapter 3](03-layers-and-displays-data-model.md)  ·  [Chapter 5 — From proto bytes to SQL tables »](05-trace-processor.md)
