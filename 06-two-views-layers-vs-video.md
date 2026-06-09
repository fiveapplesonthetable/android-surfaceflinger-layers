# Chapter 6 — Two views of one display: layers vs video

*Prerequisite: [Chapter 5](05-trace-processor.md). A short but important chapter. There are **two** Perfetto data sources that capture "the display." This one (layers) captures **structure**; the sibling (video frames) captures **pixels**. They are independent — and the one place they touch is the virtual-display trick, which is exactly why a screen recording shows up as an extra display in the layers viewer.*

This chapter cites the video side from the [android-display-video-pipeline](https://github.com/fiveapplesonthetable/android-display-video-pipeline) repo and its design docs; the layers side from live AOSP/Perfetto as in Chapters 4–5. Where the video docs disagree among themselves (they describe two wire revisions), this chapter flags it rather than guess.

---

## 6.1 The two sources at a glance

| | **Layers** (this book) | **Video frames** (sibling) |
|---|---|---|
| Data source | `android.surfaceflinger.layers` | `android.display.video` (newer) / `android.surfaceflinger.video` (earlier) |
| Produced by | SurfaceFlinger's own `LayerTracing` | `DisplayManagerService` driving a `MediaCodec` encoder |
| What it captures | the **layer tree** (per-layer bounds, transform, z, crop, color, visibility, input) | the **composited image**, encoded as H.264/HEVC |
| Granularity | N layers × geometry, **per snapshot** | **one encoded picture per on-screen change** |
| trace_processor | winscope importer → `__intrinsic_surfaceflinger_layer/_display/...` | a *separate* importer → `__intrinsic_video_frames` + a zero-copy blob fn |
| Pixels? | **No** — pure geometry | **Yes** — the actual pixels (quarter-res by default) |
| Structure? | **Yes** — every surface, stacked | **No** — already flattened by compositing |

They are **separate data sources**, separately configured (you enable each with its own `data_sources {}` block), produced by **different** subsystems, and parsed by **different** importers into **different** tables that share no storage. You can record either without the other. The only common vocabulary is the integer `display_id`.

This is the crux: **neither is derivable from the other inside the trace.** The layers source tells you *where each surface is and how it's stacked*; the video source tells you *what the screen actually looked like*. The compositor (Chapter 2) is the thing that turns the former into the latter, and that step is lossy (it discards structure).

---

## 6.2 How the video source captures — and why it needs a virtual display

The video feature can't ask the display hardware for "the final pixels" cheaply on every device. Instead it reuses the compositor: per **physical** display, it creates an **auto-mirror `VirtualDisplay`** whose output Surface *is a `MediaCodec` encoder's input surface*:

- `VirtualDisplay` created with `VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR` + `setDisplayIdToMirror(displayId)`;
- its output Surface comes from `MediaCodec.createInputSurface()`.

So SurfaceFlinger composites the mirrored screen straight into a hardware video encoder — no CPU readback. Because it's a *mirror*, it emits frames **only when the screen changes** (idle screen → no frames). Each session emits one `codec_config` packet (SPS/PPS) then one `video_frame` packet per change, carrying one Annex-B access unit; the bytes are exposed to the UI as a **zero-copy** BLOB. (Defaults: H.264, scale 0.25 — hence the small previews — key frame every ~2 s.)

Critically, the video session **skips virtual displays as capture targets** — it only ever *uses* a virtual display as its own private encoder sink, and keys its frames by the **mirrored physical `display_id`** (e.g. display 0), not the virtual display's id.

---

## 6.3 The overlap, stated precisely

Here's the subtle bit, and the one that explains the plugin's "(virtual)" displays.

When the video feature is recording, it **manufactures a virtual display** (the encoder sink) inside SurfaceFlinger. That virtual display is a real SF display: it has its own **layer stack** and (via auto-mirror) a **mirror layer** (Chapter 3.5).

Therefore, **if the layers data source is *also* recording**, that encoder virtual display becomes *observable* in the layers trace:

- it appears as a row in `__intrinsic_surfaceflinger_display` with `is_virtual = 1`;
- its mirror layer appears in `__intrinsic_surfaceflinger_layer`, on that display's layer stack.

This is purely **incidental visibility**, not a data dependency:

- The video **pixels do not come from a row in the layers snapshot.** They come out of the MediaCodec encoder, on the separate `video_frame` packets, keyed by the *source* display id.
- The two streams are not linked by id and share no storage. The encoder virtual display is simply *visible* (as `is_virtual=1`) in the layers display list, the same way any virtual display (a `screenrecord`, a Cast session, a Wi-Fi display) would be.

You can reproduce the exact same thing without the video feature at all: run `adb shell screenrecord` during a layers capture, and a `ScreenRecorder` virtual display (with a single, usually-invisible mirror layer) shows up in the layers viewer. That's the trace this book's plugin uses to test multi-display behavior (Chapter 7, Chapter 8).

---

## 6.4 Why "select a display" ≠ "view video frames"

Two different questions, two different mechanisms:

- **"Which display does this layer belong to?"** is a structural lookup: a layer's `group_id` (= its `layer_stack`) equals the display's `group_id`. Pure geometry/structure, served by the layers tables. This is what the plugin's display selector does — it re-scopes the Surface view to one display's composition (Chapter 7).
- **"What did display N actually look like at time T?"** is `__intrinsic_video_frames WHERE display_id = N` → fetch the AU bytes → WebCodecs-decode → canvas. Served by the video tables.

They don't "overlap" because they're orthogonal granularities. The layers source has N per-layer rectangles per snapshot and *no* picture; the video source has *one* encoded picture per change and *no* per-layer breakdown (compositing already flattened it). The plugin in this book is the **structure** viewer; its video sibling is the **pixels** viewer. Run both data sources and you can put them side by side on the same timeline — structure and pixels of the same frame.

---

## 6.5 The one-line summary

- **Layers** = `android.surfaceflinger.layers` → winscope importer → per-layer geometry tables, layer↔display via `group_id = layer_stack`. *(This book.)*
- **Video** = `android.display.video` → its own importer → `__intrinsic_video_frames` + zero-copy encoded blobs, captured via an auto-mirror virtual display feeding MediaCodec. *(The sibling.)*
- **Independent** data sources, configs, importers, and tables. The **only** intersection: the video feature's encoder virtual display is itself *visible* (as `is_virtual=1`, with a mirror layer) in the layers display list when both are recorded — which is precisely why the layers viewer can list a `(virtual)` display whose only layer is an invisible mirror.

Now, the plugin itself.

[« Chapter 5](05-trace-processor.md)  ·  [Chapter 7 — The UI plugin, annotated »](07-the-ui-plugin.md)
