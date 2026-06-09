# Glossary

Every recurring term, one place. Chapter references point to where it's explained in depth.

**acquire fence** — a sync token signalling "the GPU finished *writing* this buffer." Travels producer→consumer; SurfaceFlinger won't composite a layer until its acquire fence signals. (Ch. 1.5)

**ANativeWindow** — the C API apps draw into (EGL/Vulkan/Canvas target). A `Surface` is an `ANativeWindow` backed by a BufferQueue producer. (Ch. 1.2)

**arg_set_id** — a column grouping all the flattened proto fields of a row in trace_processor's generic `args` table. Lets the UI show every proto field as key/value without a column per field. (Ch. 5.1)

**base64_proto_id** — a string-pool id of a row's raw proto bytes (base64-encoded). Identical protos share one id, so comparing two layers' ids is an exact "did anything change" diff. (Ch. 5.1, 7.2)

**BLAST / BLASTBufferQueue (BBQ)** — the per-window object (in the app process) that holds the BufferQueue consumer and turns each queued frame into a SurfaceFlinger transaction carrying the buffer. (Ch. 1.6)

**buffer / GraphicBuffer** — a chunk of graphics memory (from gralloc) holding one frame's pixels. Passed by reference (slot index), never copied. (Ch. 1.2)

**BufferQueue** — the lock-protected hand-off between a producer (app) and consumer (SF): a `BufferQueueCore` plus producer (`IGraphicBufferProducer`) and consumer (`IGraphicBufferConsumer`) faces. (Ch. 1.2)

**buffer state** — a slot's state: FREE → DEQUEUED → QUEUED → ACQUIRED → FREE. (Ch. 1.2)

**client composition** — see *GPU composition*.

**commit** — SurfaceFlinger's first per-frame phase: apply queued transactions, latch buffers, rebuild the layer snapshot. Produces state, no pixels. (Ch. 2.3)

**composite** — SurfaceFlinger's second per-frame phase: turn the layer snapshot into pixels via HWC and/or the GPU. (Ch. 2.4)

**CompositionEngine (CE)** — SF's back end that performs composition (HWC + RenderEngine). (Ch. 2.1)

**Choreographer** — the app-side vsync callback dispatcher; apps schedule drawing one vsync ahead via it. (Ch. 1.5)

**dataspace** — the color space / encoding of a buffer (e.g. `STANDARD_BT709`). (Ch. 3.6)

**DEVICE composition** — see *HWC composition*.

**display frame** — where on screen a layer lands, in display pixels, after layer + display transforms. (Ch. 2.6)

**DisplayDevice** — SF's front-end wrapper for one display; owns its layer stack and a CompositionEngine `Output`. (Ch. 3.4)

**double buffering** — two buffers (`acquired + dequeued = 2`): lowest latency, but a late frame causes a dropped frame. (Ch. 1.4)

**fence** — a cheap GPU/display sync token (acquire/release/present). Lets CPU, GPU, and display run asynchronously without busy-waiting. (Ch. 1.5)

**FrontEnd** — modern SF subsystem owning the layer tree, transactions, and per-frame snapshots (`RequestedLayerState`, `LayerHierarchy`, `LayerSnapshot`). (Ch. 3.1)

**GPU / client composition** — compositing a layer by drawing it into a shared framebuffer with RenderEngine (the GPU). Flexible but costs GPU time. (Ch. 2.5)

**group_id** — in trace_processor, a rect's layer stack — the per-display grouping. Layer↔display matching is `group_id == group_id`. (Ch. 5.2)

**HWC / hardware composer / DEVICE composition** — compositing a layer via a display hardware overlay plane. Cheap/low-power but limited; the hardware can demand fallback to GPU. (Ch. 2.5)

**hwc_composition_type** — the per-layer composition type (`Composition` enum): CLIENT=1 (GPU), DEVICE=2 (HWC overlay), SOLID_COLOR=3, etc. The plugin's GPU/HWC chips. (Ch. 2.5, 4.2)

**IGBP / IGBC** — `IGraphicBufferProducer` / `IGraphicBufferConsumer`, the Binder interfaces of a BufferQueue. (Ch. 1.2)

**latching** — choosing which queued buffer to show for a layer this frame, gated on its acquire fence. (Ch. 1.5, 2.3)

**Layer** — SF's unit of composition: one content buffer plus geometry/z/state. In the trace, a fully-resolved `LayerSnapshot`. *Not* the same as an app's Surface. (Ch. 3)

**layer stack** — the integer binding layers to a display; a display composites exactly the layers whose layer stack matches its own. `UNASSIGNED` = offscreen. (Ch. 3.4)

**LayerSnapshot** — a flattened, fully-resolved, z-ordered description of one layer for one frame; the z-ordered list of these is composited and traced. (Ch. 3.1)

**LayerTracing** — SF's component that serializes a layer snapshot into the `android.surfaceflinger.layers` data source. (Ch. 4.5)

**mirror layer / mirroring** — a node reached by multiple parents so one layer's content appears on multiple displays without cloning; how screen recording shows the screen on a virtual display. (Ch. 3.5)

**Mithril** — the UI framework; components have a `view()` returning a virtual-DOM tree; re-renders + diffs on events / `m.redraw()`. (Ch. 7.0)

**present fence** — a sync token from the display hardware: "this frame actually hit the screen at time T." Feeds vsync prediction and timing. (Ch. 1.5)

**RenderEngine** — SF's GPU drawing backend (GLES/Vulkan) used for client composition. (Ch. 2.5)

**RequestedLayerState** — the server-side merged client state of one layer (transactions merge into it). (Ch. 3.1)

**release fence** — a sync token: "compositing/scan-out finished *reading* this buffer." Travels consumer→producer; the producer waits before overwriting. (Ch. 1.5)

**relative-Z / z_order_relative_of** — a layer ordered at a z relative to a *different* layer (not its parent). `z_order_relative_of` names that layer; an unreachable one is `is_missing_z_parent`. (Ch. 3.3)

**snapshot** — the entire layer tree + displays captured at one instant (one vsync). One `LayersSnapshotProto` per `TracePacket`. (Ch. 4.1)

**source crop** — which sub-rectangle of a layer's buffer is sampled during composition. (Ch. 2.6)

**SPY (input)** — an input "spy" window that observes touch without consuming it; the plugin's Spy chip (input config bit `1<<14`). (Ch. 3.6)

**Surface** — the producer-side handle an app draws into (an `ANativeWindow` over an IGBP). Distinct from a SF Layer. (Ch. 1.2)

**SurfaceFlinger (SF)** — the system process that composites all layers across all displays into the final image(s). (Ch. 2)

**trace_bounds** — the (start_ts, end_ts) the UI uses for the timeline window; computed by trace_processor by asking each importer for its extent. The fix makes the winscope importer report the snapshot tables' extent. (Ch. 5.6)

**trace_processor** — Perfetto's C++/WASM library that loads a trace and exposes SQL; the winscope importer turns layer snapshots into tables. (Ch. 5)

**TracePacket** — one record in a Perfetto trace; a layers snapshot is a `TracePacket` with field 93 set to a `LayersSnapshotProto`. (Ch. 4.6)

**transaction** — an atomic batch of layer/display changes applied on a frame boundary in commit; BLAST delivers buffers as transactions. (Ch. 2.2)

**TransformProto** — the proto encoding of a layer's affine matrix: four floats (the 2×2 linear part) + a packed `type` (mask | orientation<<8); translation is carried separately in `position`. (Ch. 4.4)

**triple buffering** — three buffers (`acquired + dequeued + 1`): the spare lets the producer not block, trading one frame of latency for fewer dropped frames. (Ch. 1.4)

**virtual display** — a display with no physical panel; its "screen" is a buffer (e.g. a video encoder, a screen recorder, a Cast sink). Appears with `is_virtual=1` in the trace. (Ch. 2.7, 3.5, 6)

**vsync** — the display's refresh heartbeat; SF wakes on it once per frame; apps schedule against it. (Ch. 1.5)

**winscope** — the standalone Android tool this plugin's capabilities derive from; also the name of the trace_processor importer and the `__intrinsic_winscope_*` tables. (Ch. 5)

[« Reviewer's guide](08-reviewers-guide.md) · [README](README.md)
