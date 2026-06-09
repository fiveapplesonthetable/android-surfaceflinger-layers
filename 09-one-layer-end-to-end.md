# Chapter 9 — One layer, end to end

*Prerequisite: Chapters 1–7. This is the capstone: a single real layer — `StatusBar#75` — traced through **every** stage with **actual numbers** pulled from the demo trace (`surfaceflinger-demo.perfetto-trace`), so the whole book collapses into one concrete thread. Nothing here is invented; every value is a real query result.*

The trace: a 5-second `MODE_ACTIVE` capture of `android.surfaceflinger.layers` on Cuttlefish. It has **76 snapshots**. We'll follow snapshot **0** (`ts = 49246683941682` ns), which has **89 layers, 4 visible**, on one physical display **`CrosvmDisplay`** (id `4619827353912518656`, `is_virtual=0`, `is_on=1`, layer-stack/`group_id` **0**).

Our subject: the **status bar** — the thin strip at the top of the screen showing the clock and icons.

---

## 9.1 On device: the status bar becomes a buffer (Chapter 1)

System UI owns a window for the status bar. Its app-side `Surface` (an `ANativeWindow` over a BufferQueue producer) is where it draws — a 720-px-wide, 48-px-tall black bar with the clock/icons composited on top. Each time the clock ticks or an icon changes, System UI:

1. **dequeues** a free buffer from its BufferQueue (`dequeueBuffer`, getting back the release fence of the previous frame),
2. **renders** the new status bar into it (GPU; an acquire fence captures completion),
3. **queues** it (`queueBuffer`, attaching the acquire fence).

Because the status bar is a normal window, that queued buffer reaches SurfaceFlinger via **BLAST** (Chapter 1.6): the window's BLASTBufferQueue acquires it and turns it into a `setBuffer` **transaction** on the status bar's `SurfaceControl`. The trace records this as the layer's current frame number — in our snapshot, `curr_frame = 570`. (The status bar had been redrawn 570 times.)

---

## 9.2 On device: SurfaceFlinger latches and composites it (Chapters 2–3)

On the next vsync, SF's **commit** phase applies that transaction and **latches** the buffer (gated on its acquire fence), then rebuilds the layer snapshot (Chapter 2.3). In that snapshot, the status bar is **layer id 75**, named `StatusBar#75`. Its resolved geometry and state (straight from the trace):

| Property | Value | Meaning (chapter) |
|---|---|---|
| `layer_stack` | **0** | belongs to display group 0 = CrosvmDisplay (Ch. 3.4) |
| `z` | 0 | its order among siblings (Ch. 3.3) |
| `bounds` | (0, 0) – (720, 48) | a full-width, 48-px-tall strip at the top (Ch. 3.6) |
| transform | identity (`dsdx=1, dsdy=1, tx=ty=0`) | no scale/rotate/translate — it sits exactly at its bounds (Ch. 4.4) |
| `color` | r=0, g=0, b=0, **a=1.0** | opaque black background (Ch. 3.6) |
| `is_opaque` | 0 | not flagged opaque (it has translucent icons over the black) |
| `dataspace` | `BT709 sRGB Full range` | the color space (Ch. 3.6) |
| `hwc_composition_type` | **`HWC_TYPE_DEVICE`** (2) | composited by a **hardware overlay**, not the GPU (Ch. 2.5) |
| `is_visible` | **1** | computed visible (Ch. 3.7) |
| `z_order_relative_of` | -1 | no relative-Z (Ch. 3.3) |
| `is_hidden_by_policy` | 0 | not hidden |
| `flags` | 8448 | SF layer flags bitfield |

Read that `hwc_composition_type = DEVICE` line in light of Chapter 2.5: SurfaceFlinger *proposed* the status bar as a DEVICE layer, HWC *validated* and accepted it (it's a simple, axis-aligned, top-of-screen rectangle — ideal for an overlay plane), so it costs **zero GPU time** to composite. A layer with a non-90° rotation, a shadow, or a colorspace HWC couldn't handle would instead show `HWC_TYPE_CLIENT` here. This one field answers "is this layer cheap to composite?", and it's exactly what the plugin's **HWC** chip surfaces.

The status bar is composited onto CrosvmDisplay because their layer stacks match (both `group_id = 0`, Chapter 3.4).

---

## 9.3 On device: the snapshot is serialized (Chapter 4)

In parallel with compositing, `LayerTracing` serializes snapshot 0 — all 89 layers plus the CrosvmDisplay — into a `LayersSnapshotProto`, and emits it as **one `TracePacket` with field 93 set** (`trace_packet.proto:231`), timestamped on `CLOCK_MONOTONIC` at `49246683941682`.

Inside that proto, layer 75 is one `LayerProto`. Using the form *fieldname (proto field number) = value* (Chapter 4.2): `id (1) = 75`, `name (2) = "StatusBar#75"`, `layer_stack (9) = 0`, `z (10) = 0`, `hwc_composition_type (35) = HWC_TYPE_DEVICE`, `color (20) = {0,0,0,1}`, `bounds (45) = {0,0,720,48}`, and a `transform (23)` whose `type` encodes IDENTITY (so its four matrix floats are omitted and default to the identity, Chapter 4.4). That's the wire form.

---

## 9.4 Off device: the proto becomes SQL rows (Chapter 5)

trace_processor's winscope importer decodes that packet. For snapshot 0 it produces:

- **one snapshot row** in `__intrinsic_surfaceflinger_layers_snapshot`: `ts = 49246683941682`, a `base64_proto_id` interning the whole snapshot's bytes, and an `arg_set_id` for the snapshot-level fields.
- **one display row** in `__intrinsic_surfaceflinger_display`: `CrosvmDisplay`, `is_virtual = 0`, with a `trace_rect_id` whose `group_id = 0`.
- **89 layer rows** in `__intrinsic_surfaceflinger_layer`, one of which is our layer 75.

For layer 75 specifically, the geometry is **factored into the dedup tables** (Chapter 5.4):

```
__intrinsic_winscope_rect       : (x=0, y=0, w=720, h=48)          ← interned, shared
__intrinsic_winscope_transform  : (dsdx=1, dtdx=0, tx=0,
                                    dtdy=0, dsdy=1, ty=0)            ← identity, shared
__intrinsic_winscope_trace_rect : { rect_id→(0,0,720,48),
                                     transform_id→identity,
                                     group_id = 0, depth = 15,
                                     is_visible = 1, opacity = 1.0 }
__intrinsic_surfaceflinger_layer: { layer_id = 75, layer_name = 'StatusBar#75',
                                     is_visible = 1, hwc_composition_type = 2,
                                     z_order_relative_of = -1, is_hidden_by_policy = 0,
                                     layer_rect_id → the trace_rect above,
                                     base64_proto_id → interned LayerProto bytes,
                                     arg_set_id = 3844 }
```

Note **`depth = 15`**: the importer walked the layers in draw order and this is the status bar's index within group 0's stack. Note **`group_id = 0`** on both the layer's trace_rect and the display — that's the join key that ties the status bar to CrosvmDisplay.

Every other proto field landed in the `args` table under `arg_set_id = 3844`. These are the real rows (queried live):

```
layer_stack = 0      z = 0           is_opaque = 0       curr_frame = 570
color.r = 0.0        color.g = 0.0   color.b = 0.0       color.a = 1.0
bounds.left = 0.0    bounds.top = 0.0  bounds.right = 720.0  bounds.bottom = 48.0
dataspace = "BT709 sRGB Full range"   hwc_composition_type = "HWC_TYPE_DEVICE"
flags = 8448         name = "StatusBar#75"
```

This is the entire contract the UI sees: typed columns for the things it queries hot (geometry, visibility, hwc, group), plus the full proto as args for the Properties pane, plus `base64_proto_id` for the exact diff.

(There's also a `visibility_reason[i]` arg family on *invisible* layers — for the 85 layers in this snapshot that aren't visible, those strings say *why*, e.g. "alpha = 0 and no blur" or "occluded". Layer 75 is visible, so it has none.)

---

## 9.5 Off device: the pixel in the plugin (Chapter 7)

You open the trace in Perfetto, expand the **SurfaceFlinger** track group, and there's one track for **CrosvmDisplay**. You jump to snapshot 0, or open the full-screen page. Here's what each pane does with layer 75:

**Surface (3D rects).** `session.displayLayers()` keeps only the layers whose `groupId === 0` (CrosvmDisplay's group) — layer 75 is in. The rects view turns its bounds into a quad by applying its transform (Chapter 7.3):
```
apply(identity, 0,   0)   = (0,   0)
apply(identity, 720, 0)   = (720, 0)
apply(identity, 720, 48)  = (720, 48)
apply(identity, 0,   48)  = (0,   48)
```
— a thin full-width strip at the very top, exactly where the status bar is on screen. Because it's `is_visible = 1` it's drawn with the visible-green fill (gradient-shaded by its depth), and a leader-line label `StatusBar#75` is placed in the right gutter. Click that label or the strip → it's selected.

**Hierarchy.** Layer 75 appears as a node `75 StatusBar#75` with chips computed from its fields (Chapter 7.6): **V** (`is_visible`), **HWC** (`hwc_composition_type = 2 ≥ 2`). No RelZ (zrel = -1), no H (not hidden), no Spy. You can hide or pin it.

**Properties.** The header card shows `StatusBar#75  #75` + the same chips. The curated **Visibility** section shows `Visible: true`. **Geometry** shows `Z: 0`, `Layer stack: 0`, `Bounds: (0, 0) – (720, 48)`, `Transform: [1 0 0] [0 1 0]` (identity). **Color & effects** shows `Color: r:0 g:0 b:0 α:1`. The proto-dump DataGrid lists every arg above; toggle **Diff** and a **Previous** column appears for whatever changed since snapshot −1 (here the clock area, so `curr_frame` and the proto's `base64_proto_id` differ — the diff marks it **modified**).

---

## 9.6 The thread, in one sentence

System UI drew a 720×48 black bar into its Surface → BLAST turned the queued buffer into a transaction → SurfaceFlinger latched it (acquire-fence gated), accepted it as a **DEVICE** (HWC overlay) layer on CrosvmDisplay (`group_id 0`), and serialized snapshot 0 → that became a `TracePacket` (field 93) → trace_processor's winscope importer flattened it into a layer row + a deduped (0,0,720,48) rect + an identity transform + an args set (`3844`) → the plugin scoped it to CrosvmDisplay, drew it as a thin top strip with a **V**+**HWC** chip, and showed its black-opaque color and bounds in the Properties pane.

Every arrow in that sentence is a chapter, and every number in this one was a real query against the trace.

[« Chapter 8](08-reviewers-guide.md)  ·  [Chapter 10 — Transactions & the other Winscope viewers »](10-transactions-and-other-viewers.md)
