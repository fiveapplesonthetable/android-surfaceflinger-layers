# SurfaceFlinger, from zero — and how to see every layer in a Perfetto trace

This is a long, deliberately-from-scratch book about two things:

1. **How Android actually puts a frame on the screen.** What an app's "Surface" really is, how pixels get from an app into a buffer, how that buffer reaches the system compositor (**SurfaceFlinger**), what double/triple buffering and vsync and fences are *for*, and how SurfaceFlinger composites many layers across one or more displays into the image you see.
2. **How that whole structure is captured into a Perfetto trace, and how a new Perfetto UI plugin (`com.android.SurfaceFlinger`) lets you inspect it** — every layer, its geometry, its z-order, what's visible, what's occluded, on which display — scrubbing frame by frame, in light or dark mode, right next to your CPU/GPU/jank tracks.

It is written for someone who knows **nothing** about Android graphics. By the end you should understand the system well enough to (a) reason about jank and composition at a professional level, and (b) read, modify, and review the plugin's code line by line.

It is the SurfaceFlinger sibling of [android-display-video-pipeline](https://github.com/fiveapplesonthetable/android-display-video-pipeline), which covered the *pixels* side (encoding the screen to video in a trace). This book covers the *structure* side (the layers), and [Chapter 6](06-two-views-layers-vs-video.md) explains exactly how the two relate.

---

## How to read this

Every claim about Android internals is backed by a citation into the **AOSP source** (`frameworks/native`, branch `android16-qpr2-release`), and every claim about the trace pipeline is backed by a citation into the **Perfetto source**. Citations look like `Layer.cpp:1475`. You do not need the source checked out to follow along — the relevant code is quoted inline — but the citations are there so you (or a reviewer) can verify nothing here is hand-waved.

> A note on line numbers: citations are against the trees at the time of writing. SurfaceFlinger and Perfetto move fast, so a given line may have drifted by a few lines by the time you read this — the quoted symbol/function is the reliable anchor; search for it if a number is slightly off.

The chapters build on each other, but each is self-contained enough to read alone:

| # | Chapter | What you'll learn |
|---|---|---|
| — | **README** (this file) | The thesis, the end-to-end map, the vocabulary you need first. |
| 1 | [Android graphics from zero](01-android-graphics-from-zero.md) | Surface, BufferQueue, producer/consumer, the buffer lifecycle, **double vs triple buffering**, vsync, latching, **fences**, BLAST. |
| 2 | [How SurfaceFlinger composites a frame](02-surfaceflinger-compositing.md) | The vsync→commit→composite loop, transactions, **HWC (overlays) vs GPU (client) composition**, composition geometry, multiple displays. |
| 3 | [The layer & display data model](03-layers-and-displays-data-model.md) | The layer tree, **z-order and relative-Z**, layer stacks, **virtual displays & mirroring** (screen recording!), geometry (bounds/crop/transform), visibility & occlusion. |
| 4 | [The layers trace](04-the-layers-trace.md) | How SurfaceFlinger emits the trace, the **proto schema** field by field, the transform-matrix encoding, the `android.surfaceflinger.layers` data source and its capture flags. |
| 5 | [From proto bytes to SQL tables](05-trace-processor.md) | Perfetto's trace_processor: the winscope importer, **how a snapshot proto becomes rows**, rect/visibility computation, `group_id`, `base64_proto_id`, and the trace-bounds fix that makes a layers-only trace open. |
| 6 | [Two views of one display: layers vs video](06-two-views-layers-vs-video.md) | The other display data source, why the two are independent, and the subtle **virtual-display** overlap. |
| 7 | [The UI plugin, annotated](07-the-ui-plugin.md) | Every file of `com.android.SurfaceFlinger`, every important snippet: the session, the data layer, the 2D-canvas 3D rects view, the hierarchy, the DataGrid properties, the timeline track. |
| 8 | [Reviewer's guide](08-reviewers-guide.md) | Invariants, the async race conditions and the token fix, memoization, the empty-state semantics, known limitations, and what to probe in review. |
| 9 | [One layer, end to end](09-one-layer-end-to-end.md) | The capstone: a single real layer (`StatusBar#75`) traced through every stage with **actual numbers** from the demo trace. |
| 10 | [Transactions & the other viewers](10-transactions-and-other-viewers.md) | The transactions data source (the *input* side), and WM / IME / ViewCapture / ProtoLog / Shell Transitions — one importer, one bounds fix. |
| 11 | [Capture it yourself](11-capture-and-explore.md) | Hands-on: capture a trace, open it, and tour every feature with screenshots. |
| — | [UI deep-dive (line-by-line)](ui-deep-dive.md) | The exhaustive companion to Ch. 7 — every algorithm in the plugin, in full. |
| — | [Glossary](glossary.md) | Every term in one place. |

If you only want the **Android graphics education**, read 1→2→3. If you only want the **trace/plugin engineering**, read 4→5→7→8. If you're reviewing the change, read 5, 7, 8.

---

## 0. The thesis

A modern phone screen is a lie told 60–120 times a second. There is no single program drawing the screen. Instead:

- Every app, the status bar, the navigation bar, the wallpaper, the soft keyboard, the on-screen cursor, a screen-recorder's mirror — each draws into **its own** off-screen buffer, independently, on its own schedule.
- A single privileged system process, **SurfaceFlinger** (SF), collects the latest buffer from each of these **layers**, figures out where each goes, who's on top, who's hidden, who's translucent, and **composites** them — ideally using the display hardware's overlay engine, falling back to the GPU when it must — into one final image per display, presented exactly at the display's refresh tick.

So the screen you see is the *output* of compositing. It throws away the structure: once layers are flattened into pixels you can no longer ask "which surface is that? how big is it? is it occluded? what's behind it?".

**The layers trace preserves that structure.** SurfaceFlinger can serialize a *snapshot* of its entire layer tree — every layer's bounds, transform, z-order, crop, color, buffer, input region, visibility, and which display it's on — once per frame, into a Perfetto trace. The `com.android.SurfaceFlinger` plugin reads those snapshots back and lets you fly through them.

This is, historically, what the standalone **Winscope** tool did. The plugin brings that capability *inside* Perfetto, so the layer structure sits on the same timeline as scheduling, CPU frequency, GPU work, frame timelines, and jank — and so you can answer "the app janked at T; what was SurfaceFlinger compositing at T, and why did that layer fall back to GPU?" in one tool. (Chapter 8 of the original Winscope tool's capabilities and the honest remaining gaps are catalogued in the reviewer's guide.)

---

## 1. The end-to-end map

Here is the entire journey of one piece of screen structure, from an app drawing a button to you clicking that button's layer in the Perfetto UI. Each step links to the chapter that explains it.

```
   ┌─────────────────────────────────────────────────────────────────────┐
   │ ON DEVICE                                                            │
   │                                                                     │
   │  App process                          SurfaceFlinger process        │
   │  ───────────                          ─────────────────────         │
   │  draw into a Surface  ──dequeue──►  ┌──────────────┐                 │
   │  (Canvas / GPU)        a buffer     │ BufferQueue  │  ── Ch.1        │
   │       │                             │ (per layer)  │                 │
   │       └──queue the buffer + fence──►└──────┬───────┘                 │
   │                                            │ acquire (latch)         │
   │                                            ▼                         │
   │                                     ┌──────────────┐                 │
   │   each vsync:  commit ──► latch ──► │  Layer tree  │  ── Ch.2, Ch.3  │
   │                                     │  (FrontEnd)  │                 │
   │                                     └──────┬───────┘                 │
   │                                            │ composite               │
   │                                            ▼                         │
   │                              ┌──────────────────────────┐            │
   │                              │ CompositionEngine        │            │
   │                              │  HWC overlays / GPU       │  ── Ch.2  │
   │                              └────────────┬─────────────┘            │
   │                                           │ present (to the panel)   │
   │                                           ▼   ... and, in parallel:  │
   │                              ┌──────────────────────────┐            │
   │   android.surfaceflinger.    │ LayerTracing             │            │
   │   layers data source  ─────► │  serialize a snapshot    │  ── Ch.4  │
   │                              │  of the whole layer tree │            │
   │                              └────────────┬─────────────┘            │
   └───────────────────────────────────────────┼────────────────────────┘
                                                │ one TracePacket per frame
                                                ▼
   ┌─────────────────────────────────────────────────────────────────────┐
   │ OFF DEVICE — the trace file, then the browser                        │
   │                                                                     │
   │  trace_processor (WASM)                                              │
   │  ──────────────────────                                             │
   │  winscope importer:  proto snapshot ──► SQL rows        ── Ch.5     │
   │    __intrinsic_surfaceflinger_layers_snapshot                        │
   │    __intrinsic_surfaceflinger_layer / _display                       │
   │    __intrinsic_winscope_trace_rect / _rect / _transform              │
   │                                            │ SQL queries             │
   │                                            ▼                         │
   │  com.android.SurfaceFlinger UI plugin                  ── Ch.7      │
   │    timeline track per display  ·  full-screen page                   │
   │    Surface (3D rects) · Hierarchy · Properties                       │
   └─────────────────────────────────────────────────────────────────────┘
```

The two on-device halves are independent: composition is what the user sees; tracing is a side-channel that serializes the same layer state SF already computed. Off device, trace_processor turns the proto into queryable tables and the plugin renders them.

---

## 2. The vocabulary you need first

You can read Chapter 1 cold, but these eight words recur everywhere, so here are one-line definitions to anchor them. The [glossary](glossary.md) has the rest.

- **Surface** — an app-side handle you draw into. Concretely, an `ANativeWindow` backed by a producer endpoint of a BufferQueue (`Surface.h:107`). *Not* the thing SF calls a layer.
- **Layer** — SurfaceFlinger's unit of composition: one piece of content plus its geometry, z-order, and state. Layers form a tree. A Surface's content becomes a layer's buffer.
- **Buffer** — a chunk of graphics memory (a `GraphicBuffer`, from gralloc) holding one frame's pixels. Passed by reference, never copied.
- **BufferQueue** — the lock-protected hand-off that lets a producer (app) and consumer (SF) pass buffers back and forth by slot index + fence, without copying pixels or sending the buffer over Binder each frame.
- **vsync** — the display's "refresh now" heartbeat (every ~16.6 ms at 60 Hz). SF wakes on it; apps schedule drawing against it via Choreographer.
- **Fence** — a cheap GPU/display synchronization token. "Tell me when the GPU finished writing this buffer" (acquire fence) / "when compositing finished reading it" (release fence) / "when it actually hit the screen" (present fence). Nobody busy-waits.
- **Composition** — combining all the visible layers of a display into one image, by hardware overlays (**HWC**) and/or the **GPU** ("client composition").
- **Layer stack** — an integer that groups layers onto a display. A display composites exactly the layers whose layer stack matches its own. This is how the same content can appear on the phone screen *and* on a screen recording (a virtual display with a different layer stack and a mirror layer).

---

## 3. A note on accuracy and honesty

Where the source is unambiguous, this book quotes it. Where something is version-specific (SurfaceFlinger has been heavily rearchitected — there is now a "FrontEnd" that owns the tree and a slim legacy `Layer` that owns the buffer pipeline), the book says so rather than pretend there's one timeless design. Where the plugin has **limitations** (things the standalone Winscope does that it doesn't yet, because they need data a layers-only trace doesn't carry), Chapter 8 lists them plainly. The goal is understanding you can trust, not marketing.

Now: [Chapter 1 — Android graphics from zero »](01-android-graphics-from-zero.md)
