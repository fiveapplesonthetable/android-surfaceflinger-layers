# Glossary

Every recurring term, one place. Chapter references point to where it's explained in depth. If you hit a term mid-chapter and it isn't defined right there, it's almost certainly here.

**acquire fence** — a sync token signalling "the GPU finished *writing* this buffer." Travels producer→consumer; SurfaceFlinger won't composite a layer until its acquire fence signals. (Ch. 1.5)

**affine transform** — a 2-D coordinate mapping that scales, rotates, shears, and translates: `(x,y) → (x',y')` via a 2×2 matrix plus an offset. A *singular* matrix is one that collapses area to zero (can't be inverted). (Ch. 3.6, 7.3)

**AIDL (Android Interface Definition Language)** — a language for declaring an interface that the build turns into cross-process (Binder) client/server stubs; e.g. the composer HAL and the `Composition` enum are AIDL. (Ch. 2.5)

**ANativeWindow** — the C API apps draw into (EGL/Vulkan/Canvas target). A `Surface` is an `ANativeWindow` backed by a BufferQueue producer. (Ch. 1.2)

**Annex-B / access unit (AU)** — H.264 terms (video sibling). An *access unit* is one coded picture's worth of bytes; *Annex-B* is the byte-stream framing that prefixes each unit with a start code so a decoder finds boundaries. (Ch. 6.2)

**arg_set_id** — a column grouping all the flattened proto fields of a row in trace_processor's generic `args` table. Lets the UI show every proto field as key/value without a column per field. (Ch. 5.1)

**args table / flattening** — trace_processor's generic key/value table. *Flattening* turns a nested proto into flat `(key, value)` rows where nested fields become dotted keys (`color.r`, `bounds.left`) and repeated fields get an index (`visibility_reason[0]`); all rows of one source share an `arg_set_id`. (Ch. 5.1)

**atrace / atrace_categories** — Android's userspace tracing tags (`gfx`, `view`, `sf`, …); enabling a category switches on the `ATRACE_*` instrumentation compiled into that subsystem, emitting named slices on the timeline. Userspace-side; distinct from ftrace. (Ch. 11.9)

**base64_proto_id** — a string-pool id of a row's raw proto bytes (base64-encoded). Identical protos share one id, so comparing two layers' ids is an exact "did anything change" diff. (Ch. 5.1, 7.2)

**Binder** — Android's kernel-backed IPC (inter-process call) mechanism; a "Binder interface" is one whose method calls cross process boundaries transparently. BufferQueue's producer/consumer faces and transaction delivery go over Binder. (Ch. 1.2, 2.2)

**BLAST / BLASTBufferQueue (BBQ)** — the per-window object (in the app process) that holds the BufferQueue consumer and turns each queued frame into a SurfaceFlinger transaction carrying the buffer. (Ch. 1.6)

**buffer / GraphicBuffer** — a chunk of graphics memory (from gralloc) holding one frame's pixels. Passed by reference (slot index), never copied. (Ch. 1.2)

**BufferQueue** — the lock-protected hand-off between a producer (app) and consumer (SF): a `BufferQueueCore` plus producer (`IGraphicBufferProducer`) and consumer (`IGraphicBufferConsumer`) faces. (Ch. 1.2)

**buffer state** — a slot's state: FREE → DEQUEUED → QUEUED → ACQUIRED → FREE. (Ch. 1.2)

**capture mode (MODE_ACTIVE / MODE_GENERATED / MODE_DUMP)** — how the layers data source decides *when* to snapshot. `MODE_ACTIVE` snapshots on every layer-state change (the book's simple capture); `MODE_GENERATED` replays SF's transaction ring buffer to reconstruct snapshots after the fact (bugreports); `MODE_DUMP` takes a single snapshot at stop. (Ch. 4.5)

**cellRenderer** — a per-column DataGrid hook returning arbitrary Mithril content for a cell (used for header rows and clickable layer references). (Ch. 7.0, 7.6)

**client composition** — see *GPU composition*.

**client target** — the single shared framebuffer RenderEngine draws all CLIENT (GPU-composited) layers into, which HWC then presents as one layer. (Ch. 2.5)

**commit** — SurfaceFlinger's first per-frame phase: apply queued transactions, latch buffers, rebuild the layer snapshot. Produces state, no pixels. (Ch. 2.3)

**composite** — SurfaceFlinger's second per-frame phase: turn the layer snapshot into pixels via HWC and/or the GPU. (Ch. 2.4)

**CompositionEngine (CE)** — SF's back end that performs composition (HWC + RenderEngine). (Ch. 2.1)

**Choreographer** — the app-side vsync callback dispatcher; apps schedule drawing one vsync ahead via it. (Ch. 1.5)

**ctx (Trace)** — the per-trace API object Perfetto hands a plugin in `onTraceLoad`; its sub-objects register UI (`pages`, `tracks`, `sidebar`), run SQL (`engine`), and drive the app (`selection`, `navigate`). (Ch. 7.0)

**dataspace** — the color space / encoding of a buffer. The proto enum reads e.g. `STANDARD_BT709`; the args table decodes it to a human label like `BT709 sRGB Full range` (what the Properties pane shows). (Ch. 3.6)

**DataGrid** — Perfetto's sortable/filterable table widget; you give it a `schema`, `columns`, and a `DataSource` (rows). The Properties pane's curated + proto-dump grids. (Ch. 7.0, 7.6)

**DEVICE composition** — see *HWC composition*.

**DispSync / vsync offsets (VSYNC-app, VSYNC-sf)** — the component that broadcasts vsync to the app and to SF at deliberately staggered offsets (app earlier, SF later) so the two pipeline stages run back-to-back; the heart of Balsini's LWN article. (Ch. 1.5)

**display frame** — where on screen a layer lands, in display pixels, after layer + display transforms. (Ch. 2.6)

**DisplayDevice** — SF's front-end wrapper for one display; owns its layer stack and a CompositionEngine `Output`. (Ch. 3.4)

**double buffering** — two buffers (`acquired + dequeued = 2`): lowest latency, but a late frame causes a dropped frame. (Ch. 1.4)

**DPR (device pixel ratio)** — `window.devicePixelRatio`: physical pixels per CSS pixel (2 on a Retina screen). The rects canvas is sized CSS-size × DPR each draw so it stays crisp. (Ch. 7.3)

**EGL** — the API that binds OpenGL ES / Vulkan rendering to a platform's native window (`ANativeWindow`); it sizes its swap chain from the BufferQueue. (Ch. 1.2)

**fence** — a cheap GPU/display sync token (acquire/release/present). Lets CPU, GPU, and display run asynchronously without busy-waiting. (Ch. 1.5)

**FK (foreign key)** — a column holding the `id` of a row in another table, linking the two (e.g. a layer's `rect_id` points into the rect table). (Ch. 5.2)

**framebuffer** — the memory the display scans out. In a naïve design apps draw here directly (causing tearing); Android avoids that with per-layer buffers + a compositor. (Ch. 1.1)

**frame timeline (`android.surfaceflinger.frametimeline`)** — the data source / tracks (Expected Timeline, Actual Timeline) showing each frame's scheduled-vs-actual present time, colored by jank type. (Ch. 11.9)

**ftrace / `linux.ftrace`** — the Linux kernel's built-in tracer; Perfetto's `linux.ftrace` data source streams selected kernel events (e.g. `sched_switch`) from its ring buffer into the trace. Kernel-side; distinct from atrace. (Ch. 11.9)

**FrontEnd** — modern SF subsystem owning the layer tree, transactions, and per-frame snapshots (`RequestedLayerState`, `LayerHierarchy`, `LayerSnapshot`). (Ch. 3.1)

**GLES (OpenGL ES)** — the embedded subset of the OpenGL GPU graphics API; one of RenderEngine's drawing backends and a way apps render into a Surface. (Ch. 1.2, 2.5)

**GPU / client composition** — compositing a layer by drawing it into a shared framebuffer with RenderEngine (the GPU). Flexible but costs GPU time. (Ch. 2.5)

**gralloc** — the graphics-memory allocator HAL; allocates `GraphicBuffer`s usable by the GPU, display, and camera at once. (See *buffer / GraphicBuffer*, Ch. 1.2)

**group_id** — in trace_processor, a rect's layer stack — the per-display grouping. Layer↔display matching is `group_id == group_id`. (Ch. 5.2)

**HAL (Hardware Abstraction Layer)** — the standardized (C / AIDL) interface through which Android talks to a vendor's hardware driver; SF reaches the display through the *composer HAL* (HWC). (Ch. 2)

**HWC / HWC2 / hardware composer / DEVICE composition** — the display's dedicated compositing hardware, driven through the composer HAL (HWC2 = version 2 of that AIDL interface). Composes a layer via an *overlay plane*: cheap/low-power but limited; the hardware can demand fallback to GPU. (Ch. 2.5)

**hwc_composition_type** — the per-layer composition type in the *trace* (proto enum `HwcCompositionType`: `HWC_TYPE_CLIENT=1` (GPU), `HWC_TYPE_DEVICE=2` (HWC overlay), `HWC_TYPE_SOLID_COLOR=3`, …). It mirrors the AIDL `Composition` enum SF uses internally. The plugin's GPU/HWC chips. (Ch. 2.5, 4.2)

**IGBP / IGBC** — `IGraphicBufferProducer` / `IGraphicBufferConsumer`, the Binder interfaces of a BufferQueue. (Ch. 1.2)

**importer** — a trace_processor module that decodes one kind of trace packet and inserts its contents as rows into SQL tables. The *winscope importer* handles the layer snapshots. (Ch. 5.1)

**interning / string pool** — *interning* stores each distinct value once in a shared table (the *string pool*) and hands out a small integer id; equal values get the same id, so you compare ids instead of bytes. The basis of `base64_proto_id`'s exact diff. (Ch. 5.1)

**intrinsic table** — trace_processor's built-in tables, declared in Python and code-generated into C++ classes and SQL names (all prefixed `__intrinsic_`); the lowest-level tables the importer writes into, as opposed to SQL views layered on top. (Ch. 5.1)

**jank** — a visible stutter: a frame missed its display deadline (presented late or was dropped), so the previous frame is shown again. The frame timeline classifies *why* (app late, SF late, buffer stuffing, …). (Ch. 1.5, 11.9)

**latching** — choosing which queued buffer to show for a layer this frame, gated on its acquire fence. (Ch. 1.5, 2.3)

**Layer** — SF's unit of composition: one content buffer plus geometry/z/state. In the trace, a fully-resolved `LayerSnapshot`. *Not* the same as an app's Surface. (Ch. 3)

**layer stack** — the integer binding layers to a display; a display composites exactly the layers whose layer stack matches its own. `UNASSIGNED` = offscreen. (Ch. 3.4)

**LayerSnapshot** — a flattened, fully-resolved, z-ordered description of one layer for one frame; the z-ordered list of these is composited and traced. (Ch. 3.1)

**LayerTracing** — SF's component that serializes a layer snapshot into the `android.surfaceflinger.layers` data source. (Ch. 4.5)

**Looper** — Android's per-thread event loop; it blocks until a message or callback (here, a vsync) is posted, then runs it. SF's main thread runs on one. (Ch. 2.1)

**MediaCodec** — Android's low-level API for hardware-accelerated video encode/decode; the video data source feeds a virtual display's output into a `MediaCodec` H.264 encoder's input Surface. (Ch. 6.2)

**memoization** — caching a computed result and reusing it while its inputs are unchanged (compared by `===` identity). The DataGrid data sources are memoized so a redraw doesn't discard the grid's sort/filter state. (Ch. 7.6)

**mirror layer / mirroring** — a node reached by multiple parents so one layer's content appears on multiple displays without cloning; how screen recording shows the screen on a virtual display. `AUTO_MIRROR` is the virtual-display flag that makes the OS do this automatically. (Ch. 3.5, 6.2)

**Mithril** — the UI framework. `m('tag.class', attrs, children)` builds a virtual-DOM node (the string is a CSS-selector-like tag); a component's `view()` returns that tree and re-runs on events or `m.redraw()`; `oncreate`/`onupdate` are lifecycle hooks, `v.dom` is the real DOM element. (Ch. 7.0)

**onTraceLoad** — a plugin's lifecycle entry point: Perfetto calls it once, after a trace is parsed, passing the `Trace` context (`ctx`). Everything the plugin sets up happens here. (Ch. 7.0)

**orthographic projection** — flattening 3-D to 2-D by ignoring depth (no perspective foreshortening) — here literally dropping the Z coordinate after rotation, so parallel layers stay parallel. (Ch. 7.3)

**overlay plane** — a dedicated display-hardware compositing layer that scans a buffer out directly, no GPU; cheap and low-power but limited in count and capability. What HWC/DEVICE composition uses. (Ch. 2.5)

**PerfettoPlugin** — the interface a plugin class implements (with a `static readonly id`); it runs only if listed in `default_plugins.ts`, the app's allow-list. (Ch. 7.0)

**present fence** — a sync token from the display hardware: "this frame actually hit the screen at time T." Feeds vsync prediction and timing. (Ch. 1.5)

**RenderEngine** — SF's GPU drawing backend (GLES/Vulkan) used for client composition. (Ch. 2.5)

**RequestedLayerState** — the server-side merged client state of one layer (transactions merge into it). (Ch. 3.1)

**release fence** — a sync token: "compositing/scan-out finished *reading* this buffer." Travels consumer→producer; the producer waits before overwriting. (Ch. 1.5)

**relative-Z / z_order_relative_of** — a layer ordered at a z relative to a *different* layer (not its parent). `z_order_relative_of` names that layer; an unreachable one is `is_missing_z_parent`. (Ch. 3.3)

**RAII / `sp<T>`** — `sp<T>` is Android's reference-counted smart pointer (a RAII handle, like `std::shared_ptr`): the object is freed when the last `sp` to it goes away. (Ch. 1.2)

**registerPage / registerTrack** — Perfetto plugin calls: `registerPage` adds a full-screen view at a URL route; `registerTrack` makes a track available under a unique `uri` (its address) that other code selects by. (Ch. 7.7)

**scan-out** — the display hardware reading a finished buffer pixel-by-pixel and driving it onto the panel. (Ch. 1.1)

**SCHED_DEADLINE** — a Linux real-time scheduling policy ("run this much work by this time") that SF / RenderEngine threads can use to reliably hit vsync deadlines; covered by Balsini's LWN article. (Ch. 1.5)

**sched_switch** — the kernel ftrace event emitted on every CPU context switch (old task → new task); the basis for Perfetto's per-thread scheduling tracks. (Ch. 11.9)

**slice** — a single time-bar on a Perfetto track (start `ts`, duration `dur`); `dur = -1` marks an incomplete slice drawn as an instant. `depth` is its stacking row. (Ch. 7.8)

**snapshot** — the entire layer tree + displays captured at one instant (one vsync). One `LayersSnapshotProto` per `TracePacket`. (Ch. 4.1)

**source crop** — which sub-rectangle of a layer's buffer is sampled during composition. (Ch. 2.6)

**SPS / PPS** — H.264 Sequence / Picture Parameter Sets: small headers (resolution, profile, …) a decoder needs once up front; they ride in a `codec_config` packet. (Ch. 6.2)

**SPY (input)** — an input "spy" window that observes touch without consuming it; the plugin's Spy chip (input config bit `1<<14`). (Ch. 3.6)

**Surface** — the producer-side handle an app draws into (an `ANativeWindow` over an IGBP). Distinct from a SF Layer. (Ch. 1.2)

**SurfaceControl** — the app-side handle to one SF layer; transactions (`setBuffer`, `setPosition`, …) are applied to a `SurfaceControl`. (Ch. 1.6)

**SurfaceFlinger (SF)** — the system process that composites all layers across all displays into the final image(s). (Ch. 2)

**token (race guard)** — a monotonic (only-increasing) counter bumped per load/selection; an async operation captures the current value and, after each `await`, discards its result if a newer operation has bumped the counter (i.e. it was *superseded*). How the plugin avoids races. (Ch. 7.2)

**trace_bounds** — the (start_ts, end_ts) the UI uses for the timeline window; computed by trace_processor by asking each importer for its extent. The fix makes the winscope importer report the snapshot tables' extent. (Ch. 5.6)

**trace_processor** — Perfetto's C++/WASM library that loads a trace and exposes SQL; the winscope importer turns layer snapshots into tables. (Ch. 5)

**trace flags (TRACE_FLAG_COMPOSITION / INPUT / VIRTUAL_DISPLAYS / …)** — per-capture bits selecting what each snapshot carries: `COMPOSITION` adds the visible region + `hwc_composition_type` (the GPU/HWC chips); `INPUT` adds `input_window_info` (the Input section + Spy chip); `VIRTUAL_DISPLAYS` includes virtual-display layers. (Ch. 4.5)

**TracePacket** — one record in a Perfetto trace; a layers snapshot is a `TracePacket` with field 93 set to a `LayersSnapshotProto`. (Ch. 4.6)

**TrackEventDetailsPanel** — the panel Perfetto shows in the bottom drawer when a slice is selected; the framework calls its `load(selection)` then `render()`. (Ch. 7.8)

**transaction** — an atomic batch of layer/display changes applied on a frame boundary in commit; BLAST delivers buffers as transactions. (Ch. 2.2)

**TransformProto** — the proto encoding of a layer's affine matrix: four floats (the 2×2 linear part) + a packed `type` (mask | orientation<<8); translation is carried separately in `position`. (Ch. 4.4)

**triple buffering** — three buffers (`acquired + dequeued + 1`): the spare lets the producer not block, trading one frame of latency for fewer dropped frames. (Ch. 1.4)

**virtual display** — a display with no physical panel; its "screen" is a buffer (e.g. a video encoder, a screen recorder, a Cast sink). Appears with `is_virtual=1` in the trace. (Ch. 2.7, 3.5, 6)

**vsync** — the display's refresh heartbeat; SF wakes on it once per frame; apps schedule against it. The *vsync deadline* is the moment a frame's work must finish to present on the next refresh; the *refresh period* is the time between vsyncs (16.6 ms at 60 Hz). (Ch. 1.5)

**WASM (WebAssembly)** — a portable binary format that lets the C++ `trace_processor` run inside the browser, so the web UI queries traces locally with no server (each query is an async round-trip to it). (Ch. 5)

**winscope** — the standalone Android tool this plugin's capabilities derive from; also the name of the trace_processor importer and the `__intrinsic_winscope_*` tables. (Ch. 5)

**yaw / pitch** — *yaw* is rotation about the vertical (Y) axis (turning left/right); *pitch* is rotation about the horizontal (X) axis (tilting up/down). Together they tilt the flat layer stack into the 3-D look. (Ch. 7.3)

[« Reviewer's guide](08-reviewers-guide.md) · [README](README.md)
