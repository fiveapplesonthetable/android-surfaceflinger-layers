# Chapter 2 — How SurfaceFlinger composites a frame

*Prerequisite: [Chapter 1](01-android-graphics-from-zero.md). Now that one layer's buffer has reached SurfaceFlinger, we follow what SF does with *all* the layers, every vsync: the two-phase loop, what a "transaction" is, and the central decision — hardware overlays (HWC) vs the GPU (client composition).*

AOSP branch `android16-qpr2-release`; paths under `frameworks/native/services/surfaceflinger/`.

SurfaceFlinger has two halves you must keep separate:

- a **front end** — the layer tree, transactions, and the per-frame *snapshot* of layer state (this is what gets traced; Chapters 3–4);
- a **back end** called **CompositionEngine (CE)** — turns that state into actual pixels via the **HWC HAL** and/or **RenderEngine (GPU)**.

The front end decides *what* the screen should look like; CE decides *how* to produce it.

---

## 2.1 The loop: vsync → commit → composite

SurfaceFlinger runs a single main thread on a `Looper`. It does not busy-loop; the `Scheduler` wakes it once per vsync. When it wakes it runs exactly two steps, in order: **`commit()`** then (if needed) **`composite()`**. The contract is the `ICompositor` interface:

```cpp
// Scheduler/include/scheduler/interface/ICompositor.h:37
struct ICompositor {
    virtual void configure() = 0;
    // Commits transactions ... Returns whether any state has been invalidated,
    // i.e. whether a frame should be composited for each display.
    virtual bool commit(PhysicalDisplayId pacesetterId, const scheduler::FrameTargets&) = 0;
    // Composites a frame for each display. CompositionEngine performs GPU and/or
    // HAL composition via RenderEngine and the Composer HAL, respectively.
    virtual CompositeResultsPerDisplay composite(PhysicalDisplayId pacesetterId,
                                                 const scheduler::FrameTargeters&) = 0;
};
```

`SurfaceFlinger` *is* the `ICompositor`. The wake path is: the display's vsync fires `MessageQueue::vsyncCallback` → `dispatchFrame` posts a message to the looper → `handleMessage` calls `Scheduler::onFrameSignal`, which drives the two phases:

```cpp
// Scheduler/Scheduler.cpp:475
        if (!compositor.commit(pacesetterPtr->displayId, targets)) {
            ...
            return;                               // commit said "nothing to draw" → stop
        }
    }
    ...
    const auto resultsPerDisplay = compositor.composite(pacesetterPtr->displayId, targeters);
```

("pacesetter" = the display that sets the cadence; other displays are timed relative to it.)

---

## 2.2 What a transaction is

A **transaction** is an atomic batch of layer/display changes assembled on the client side and applied all at once on a frame boundary. The client class is `SurfaceComposerClient::Transaction` (`libs/gui/include/gui/SurfaceComposerClient.h:465`), with setters like `setPosition`, `setLayer` (the Z order), `setLayerStack`, `setBuffer`. Each setter mutates a `layer_state_t` and records a dirty bit in a `what` mask:

```cpp
// libs/gui/SurfaceComposerClient.cpp:1359 (setLayer = absolute Z)
    s->what |= layer_state_t::eLayerChanged;
    s->what &= ~layer_state_t::eRelativeLayerChanged;
    s->z = z;
```

`Transaction::apply()` ships the batch to SF over Binder (`SurfaceComposerClient.cpp:1201`, `sf->setTransactionState(...)`). On the SF side it is **not applied immediately** — it's resolved, wrapped in a `QueuedTransactionState`, pushed onto a lockless queue, and a commit is scheduled:

```cpp
// SurfaceFlinger.cpp:5499
        mTransactionHandler.queueTransaction(std::move(state));
    }
    ...
    setTransactionFlags(eTransactionFlushNeeded, schedule, frameHint, ...);
```

So a transaction takes effect on the next **commit**, on a frame boundary — that's what makes a multi-property change (move + resize + restack) atomic: the user never sees a half-applied state. This is also the mechanism BLAST uses to deliver an app's buffer (Chapter 1.6): `setBuffer` is just another transaction field.

---

## 2.3 The commit phase (front end)

`SurfaceFlinger::commit()` decides whether anything changed enough to require drawing. Its heart is `updateLayerSnapshots`, and its return value becomes `mustComposite`:

```cpp
// SurfaceFlinger.cpp:2950
        const bool flushTransactions = clearTransactionFlags(eTransactionFlushNeeded);
        mustComposite |= updateLayerSnapshots(vsyncId, ..., expectedPresentTime, flushTransactions, ...);
```

Inside `updateLayerSnapshots` three things happen, in order:

1. **Drain & apply queued transactions** to the layer lifecycle and hierarchy:
   ```cpp
   // SurfaceFlinger.cpp:2680
       update.transactions = mTransactionHandler.flushTransactions();
       mLayerLifecycleManager.applyTransactions(update.transactions);
       ...
       mLayerHierarchyBuilder.update(mLayerLifecycleManager);
   ```
2. **Latch buffers** — bind each layer's newest client-submitted buffer for this frame (gated on its acquire fence, Chapter 1.5):
   ```cpp
   // SurfaceFlinger.cpp:2802
       it->second->latchBufferImpl(unused, latchTime, expectedPresentTimeNs, bgColorOnly);
       newDataLatched = true;
   ```
3. **Rebuild the layer snapshots** — the immutable, fully-resolved, z-ordered per-frame view of every layer (`SurfaceFlinger.cpp:2733`, `mLayerSnapshotBuilder.update(args)`). **This snapshot tree is exactly what the layers trace serializes** (Chapter 4).

`commit()` returns `mustComposite` (`SurfaceFlinger.cpp:3007`). If nothing visible changed, `composite()` is skipped entirely — SF does no work for a static screen.

**Commit produces no pixels.** It produces *state*.

---

## 2.4 The composite phase (back end)

`SurfaceFlinger::composite()` (`SurfaceFlinger.cpp:3118`) assembles a `CompositionRefreshArgs` — the per-frame inputs: the list of output displays, dirty/geometry flags, color settings, the layer snapshots — and hands it to CompositionEngine:

```cpp
// SurfaceFlinger.cpp:3233
    mCompositionEngine->present(mainThreadRefreshArgs);
```

CE loops over every output (display) and runs the pipeline per output:

```cpp
// CompositionEngine/src/CompositionEngine.cpp:130
void CompositionEngine::present(CompositionRefreshArgs& args) {
    preComposition(args);
    for (const auto& output : args.outputs) { output->prepare(args, latchedLayers); }
    offloadOutputs(args.outputs);
    for (const auto& output : args.outputs) { presentFutures.push_back(output->present(args)); }
    ...
    postComposition(args);
}
```

`Output::present` (`Output.cpp:474`) is the per-display sequence; the three steps that matter are:

- `prepareFrame()` — **choose** HWC vs client per layer (validate with HWC),
- `finishFrame()` — **GPU-render** any client layers,
- `presentFrameAndReleaseLayers()` — **HWC present** + collect release fences.

---

## 2.5 The central decision: HWC overlays vs GPU (client) composition

This is the most important idea in compositing. For each layer, each frame, SF can composite it one of two ways:

- **HWC / DEVICE composition** — the display hardware's compositor places the buffer directly as an **overlay**. Cheap, low-power, but the hardware has a limited number of overlay planes and capabilities (scaling, rotation, blending, color transforms it can't all do).
- **Client / GPU composition** — SF uses **RenderEngine** to draw the layer into a single shared "client target" framebuffer. Flexible (the GPU can do anything), but costs GPU time and power.

The hardware gets the final say. The dance is **propose → validate → fall back**:

1. SF proposes that layers be DEVICE-composited and writes their state to HWC2 layers.
2. SF calls **validateDisplay**; HWC returns **getChangedCompositionTypes** — the layers it *demands* be done as CLIENT instead (because it ran out of planes or can't handle that layer).
3. SF GPU-renders exactly those layers into the client target, hands that framebuffer to HWC as one CLIENT layer, and HWC presents.

The validate/getChanges sequence:

```cpp
// DisplayHardware/HWComposer.cpp:582 (validate or fast-present)
        error = hwcDisplay->validate(expectedPresentTime, ..., &numTypes, &numRequests);
// :606 (read back the demanded changes)
    error = hwcDisplay->getChangedCompositionTypes(&changedTypes);
    ...
    error = hwcDisplay->getClientTargetProperty(&clientTargetProperty);
    ...
    error = hwcDisplay->acceptChanges();
```

There's an optimization — if no layer already needs the GPU, SF may **skip validate** and present in one call (the "fast path", `HWComposer.cpp:560`). The moment any layer requires client composition, validate is mandatory.

Applying the result marks the demanded layers CLIENT and records what kind of composition the frame uses:

```cpp
// CompositionEngine/src/Display.cpp:295
void Display::applyCompositionStrategy(const std::optional<DeviceRequestedChanges>& changes) {
    if (changes) {
        applyChangedTypesToLayers(changes->changedTypes);   // "this layer must be CLIENT"
        ...
    }
    state.usesClientComposition = anyLayersRequireClientComposition();
    state.usesDeviceComposition = !allLayersRequireClientComposition();
}
```

### Two ways a layer ends up on the GPU

1. **HWC demanded it** — it came back in `changedTypes`. A layer "requires client composition" iff its HWC type is CLIENT (or it has no HWC layer):
   ```cpp
   // CompositionEngine/src/OutputLayer.cpp:928
   bool OutputLayer::requiresClientComposition() const {
       const auto& state = getState();
       return !state.hwc || state.hwc->hwcCompositionType == Composition::CLIENT;
   }
   ```
2. **SF forced it** (`forceClientComposition`) — SF itself knows HWC can't (or shouldn't) overlay this layer. Reasons include a **secure layer on a non-secure output**, an **invalid (non-90°) rotation** (`OutputLayer.cpp:359`), a **dataspace HWC can't handle**, or a **per-layer color transform HWC rejected** (`OutputLayer.cpp:443`, `:712`). Shadows and some effects also force it.

### The GPU render

`Output::finishFrame` dequeues the client-target buffer only if needed and calls `composeSurfaces`, which short-circuits when there's nothing to client-composite, else builds per-layer `LayerSettings` and calls RenderEngine:

```cpp
// CompositionEngine/src/Output.cpp:1454
    auto fenceResult = renderEngine
                           .drawLayers(clientCompositionDisplay, clientRenderEngineLayers, tex, std::move(fd))
                           .get();
```
There's a cache: an identical client-composition request reuses the previous buffer instead of re-rendering (`Output.cpp:1415`).

### `hwcCompositionType` — the values you'll see in the trace

This per-layer type is the AIDL enum `Composition`, backed by `int`. **You will see these exact integers in the layer table's `hwc_composition_type` column** (Chapter 5), and the UI turns them into the `GPU`/`HWC` chips (Chapter 7):

```cpp
// hardware/interfaces/graphics/composer/aidl/.../composer3/Composition.aidl:22
enum Composition {
    INVALID = 0,
    CLIENT = 1,               // GPU renders it into the client-target buffer
    DEVICE = 2,               // HWC overlay; may be downgraded to CLIENT at validate
    SOLID_COLOR = 3,          // HWC fills a color
    CURSOR = 4,               // like DEVICE, position settable asynchronously
    SIDEBAND = 5,             // HWC pulls buffer updates itself (e.g. video)
    DISPLAY_DECORATION = 6,   // cutout / rounded-corner overlay
    REFRESH_RATE_INDICATOR = 7,
}
```

(The plugin's chip logic: `hwcCompositionType === 1` → a yellow **GPU** chip, `>= 2` → a blue **HWC** chip. See Chapter 7.)

---

## 2.6 Composition geometry — how a layer lands on screen

Each layer carries a buffer (in *buffer space*), a window crop, a per-layer transform (the scale/rotate/translate WindowManager applied), and the display's global transform (screen rotation + scaling to the panel). Composition reduces these to two rectangles + an orientation per layer:

- the **display frame** — *where on screen* the layer lands, in display pixels;
- the **source crop** — *which sub-rectangle of the buffer* is sampled;
- the **buffer transform** — orientation only (the rotation/flip), since scale/translate are baked into the two rectangles.

The display frame is layer-space → layerStack-space → display-space:

```cpp
// CompositionEngine/src/OutputLayer.cpp:190
Rect OutputLayer::calculateOutputDisplayFrame() const {
    ...
    geomLayerBounds = layerTransform.transform(geomLayerBounds);     // apply WM transform
    FloatRect frame = reduce(geomLayerBounds, activeTransparentRegion);
    frame = frame.intersect(outputState.layerStackSpace.getContent().toFloatRect());  // clip to viewport
    const ui::Transform displayTransform{outputState.transform};
    return Rect(displayTransform.transform(frame));                  // apply display transform
}
```

The transforms compose in a specific order (the source comment is gold):

```cpp
// CompositionEngine/src/OutputLayer.cpp:251
    /*
     * Transformations are applied in this order:
     * 1) buffer orientation/flip/mirror
     * 2) state transformation (window manager)
     * 3) layer orientation (screen orientation)
     * (NOTE: the matrices are multiplied in reverse order)
     */
    ui::Transform transform(displayTransform * layerTransform * bufferTransform);
```

This is the same affine-transform idea the plugin uses when it draws each layer's rect (Chapter 7), and the same matrix the trace serializes (Chapter 4's `TransformProto`).

---

## 2.7 Multiple displays and where virtual displays fit

Each display SF drives has a `DisplayDevice` owning a CompositionEngine `Output`. Every layer is tagged with a **layer stack** id; a display composites only the layers whose layer stack matches its own, via a `LayerFilter`:

```cpp
// services/surfaceflinger/common/include/common/LayerFilter.h
    bool includes(LayerFilter other) const {
        if (other.layerStack == ui::UNASSIGNED_LAYER_STACK || other.layerStack != layerStack) {
            return false;   // the layer stacks must match
        }
        ...
    }
```

SF iterates all outputs and composites each independently (`CompositionEngine::present` loop above). Two displays cannot composite the *same* layer stack (de-dup at `SurfaceFlinger.cpp:3034`).

A **virtual display** is just an `Output` with no physical panel — its "screen" is a buffer (a `BufferQueue`/Surface) someone provided. A GPU-backed virtual display renders via RenderEngine into that buffer:

```cpp
// CompositionEngine/src/Display.cpp:89
bool Display::isVirtual() const {
    return !std::holds_alternative<PhysicalDisplayId>(mIdVariant);
}
```

GPU virtual displays can even be **offloaded** to a background CompositionEngine so they don't stall the main thread (`SurfaceFlinger.cpp:3083`). This is exactly the mechanism behind screen recording and casting — and it's why a recorder shows up as an extra display in the trace. Chapter 3 explains how a virtual display gets its content (mirroring) and Chapter 6 connects it to the video data source.

---

## 2.7b The whole pipeline, drawn

```
vsync ─► Scheduler::onFrameSignal
          │
          ├─ commit() ── flush transactions ─► apply to layer tree ─► latch buffers
          │              ─► rebuild LayerSnapshot ─► returns mustComposite
          │                         └──────────────► LayerTracing serializes it ──► trace (Ch.4)
          │
          └─ composite() ── CompositionEngine::present, per display Output:
                              prepareFrame ─► validate with HWC ─► getChangedCompositionTypes
                                  │                                       │
                            DEVICE layers                          CLIENT layers
                                  │                                       │
                            HWC overlay plane                  RenderEngine (GPU) ─► client target
                                  └───────────────┬───────────────────────┘
                                                  ▼
                                        present (to the panel) ─► present fence
```

## 2.8 The whole frame, in one breath

1. vsync → `Scheduler::onFrameSignal`.
2. **commit:** flush transactions → apply to the layer tree → latch each layer's buffer (acquire-fence gated) → rebuild the layer **snapshot** → return `mustComposite`.
3. if `mustComposite`, **composite:** build `CompositionRefreshArgs` → `CompositionEngine::present` → for each display: choose HWC vs GPU (validate → getChangedCompositionTypes), GPU-render the CLIENT layers, HWC-present, collect release + present fences.
4. (in parallel, a side-channel) **LayerTracing** serializes the snapshot from step 2 into the trace — next chapter is what's *in* that snapshot.

[« Chapter 1](01-android-graphics-from-zero.md)  ·  [Chapter 3 — The layer & display data model »](03-layers-and-displays-data-model.md)
