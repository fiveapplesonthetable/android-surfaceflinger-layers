# Chapter 3 — The layer & display data model

*Prerequisite: [Chapter 2](02-surfaceflinger-compositing.md). This is the chapter that maps one-to-one onto what you'll see in the Perfetto UI: the layer tree, z-order and relative-Z, layer stacks, virtual displays and **mirroring** (how screen recording works), per-layer geometry, and how SF decides a layer is visible or occluded.*

AOSP `android16-qpr2-release`; paths under `frameworks/native/services/surfaceflinger/`.

---

## 3.1 Two representations of a "layer"

Modern SurfaceFlinger has been split in two, and the code looks contradictory until you know why. The old `Layer` was one god-object: it held the tree, the geometry, the z-order resolution, *and* each layer's buffer pipeline — so every transaction touched a tangle of shared mutable state. The FrontEnd rewrite pulled the *data model* (tree, state, per-frame snapshot) out into its own immutable-snapshot pipeline, leaving `Layer` to own only the buffer plumbing. So:

1. **The legacy `Layer` object** (`Layer.h`/`Layer.cpp`) is now *slim*. It owns the per-layer **buffer pipeline** (acquire fences, release callbacks, frame timeline, buffer latching) and a `State mDrawingState`. It no longer owns the tree, z-order, or geometry resolution.
2. **The "FrontEnd"** (`FrontEnd/`) owns the data model this chapter is about. Its readme states the design:
   > "SurfaceFlinger FrontEnd implements the client APIs that describe how buffers should be composited on the screen. Layers are used to capture how the buffer should be composited and each buffer is associated with a Layer." — `FrontEnd/readme.md:3`

The FrontEnd has three data classes; you will meet all three:

- **`RequestedLayerState`** (`FrontEnd/RequestedLayerState.h:36`, `: layer_state_t`) — the server-side *merged client state* of one layer. Transactions merge into it.
- **`LayerHierarchy`** (`FrontEnd/LayerHierarchy.h:44`) — the tree/graph, built from the requested states.
- **`LayerSnapshot`** (`FrontEnd/LayerSnapshot.h:63`, `: public LayerFECompositionState`) — a **flattened, fully-resolved, z-ordered** description of one layer for one frame. The z-ordered list of these snapshots is what gets composited (Chapter 2) **and what gets traced** (Chapter 4).

When this book says "a layer" in the trace, it means a `LayerSnapshot`.

---

## 3.2 The layer tree (really a graph)

A layer describes how one buffer is placed relative to others:

> "Layers are used to describe how a buffer should be placed on the display relative to other buffers. They are represented as a hierarchy, similar to a scene graph. Child layers can inherit some properties from their parents." — `FrontEnd/readme.md:13`

It is a **graph**, not a strict tree, because a node can be reached via multiple parents (this is how mirroring works without cloning — §3.5):

```cpp
// FrontEnd/LayerHierarchy.h:46 — the edge kinds
    Attached,         // child of the parent
    Detached,         // child of the parent but currently relative-parented elsewhere
    Relative,         // relative child of the parent
    Mirror,           // mirrored from another layer
    Detached_Mirror,  // mirrored, ignoring the source's local transform
```

The requested state holds **ids, not pointers**, so layer handles don't get their lifetime extended:

```cpp
// FrontEnd/RequestedLayerState.h:127
    uint32_t parentId = UNASSIGNED_LAYER_ID;
    uint32_t relativeParentId = UNASSIGNED_LAYER_ID;
    uint32_t layerIdToMirror = UNASSIGNED_LAYER_ID;
```

There are **two roots**: an onscreen root and an `mOffscreenRoot` (`LayerHierarchy.h:234`). A layer with no parent that "can't be root" goes offscreen — its handle is alive but it isn't composited. That's the lifecycle rule:

> "If the layer is not reachable but its handle is alive, the layer will be offscreen and its resources will not be freed." — `FrontEnd/readme.md:43`

Each layer also has a stable `sequence`/`id` used as the final tiebreaker for sort order (`Layer.h:366`), so two layers with the same stack and z still have a deterministic order (newer on top).

---

## 3.3 Z-order and relative-Z

### Absolute z

`z` is an `int32_t` on the client state (`gui/LayerState.h:392`). `setLayer(z)` sets it and clears any relative-Z (the snippet in Chapter 2.2). The drawing order:

> "Layers are drawn based on an inorder traversal, treating relative parents as direct parents. Negative z-values place layers below their parent, while non-negative values place them above. Layers with the same z-value are drawn in creation order (newer on top)." — `FrontEnd/readme.md:20`

The actual comparator — note the precedence **layer stack, then z, then id**:

```cpp
// FrontEnd/LayerHierarchy.cpp:28
    if (lhsLayer->layerStack.id != rhsLayer->layerStack.id) return lhsLayer->layerStack.id < rhsLayer->layerStack.id;
    if (lhsLayer->z != rhsLayer->z)                         return lhsLayer->z < rhsLayer->z;
    return lhsLayer->id < rhsLayer->id;
```

Flattening that traversal top-to-bottom gives each layer a **draw index** within its layer stack — its position in paint order, which is distinct from its `z` value. The trace records this as each rect's `depth` (Chapter 5), and the plugin stacks the 3D rects along it (Chapter 7).

### Relative-Z

`setRelativeLayer(relativeTo, z)` makes a layer sit at z **relative to a different layer**, not its parent. In the graph the layer becomes a child of *two* nodes: its real parent (now marked `Detached`, so normal traversal skips it) and its relative parent (added as `Relative`):

```cpp
// FrontEnd/LayerHierarchy.cpp:262 (attachToRelativeParent)
    hierarchy->mRelativeParent->addChild(hierarchy, LayerHierarchy::Variant::Relative);
    hierarchy->mParent->updateChild(hierarchy, LayerHierarchy::Variant::Detached);
```

The trace field for this is **`z_order_relative_of`** — the id of the relative parent, or `-1` if none (`LayerProtoHelper.cpp:301`). The plugin shows a **RelZ** chip when `z_order_relative_of > 0`, and a **RelZParent** chip on the layer that *others* are relative to (Chapter 7).

### "Missing relative-Z parent" and reachability

If a layer is relatively-parented to something that doesn't exist on-screen, it's unreachable through the normal tree. Modern SF tracks this as **Reachability**:

```cpp
// FrontEnd/LayerSnapshot.h:125
enum class Reachability : uint32_t {
    Reachable,
    Unreachable,
    ReachableByRelativeParent,   // only reachable via a relative parent
};
```

The trace surfaces a boolean `is_missing_z_parent`, which the importer sets when `z_order_relative_of > 0` but that layer id isn't in the snapshot (Chapter 5). The plugin shows a red **MissingZ** chip. SF even detects and breaks **relative-Z loops** (A relative-to-B, B relative-to-A) — `LayerHierarchy.cpp:444`.

---

## 3.4 Layer stacks and displays

A **layer stack** is the integer that binds layers to a display:

> "A LayerStack identifies a Z-ordered group of layers. A layer can only be associated to a single LayerStack, and a LayerStack should be unique to each display in the composition target." — `ui/LayerStack.h:27`

`LayerStack{0}` is the default; `UNASSIGNED_LAYER_STACK` means **offscreen** (`ui/LayerStack.h:43`). A layer's effective layer stack is **inherited from its root ancestor**, set only at the root:

```cpp
// FrontEnd/LayerSnapshotBuilder.cpp:770
    snapshot.outputFilter.layerStack =
        parentSnapshot.path == LayerHierarchy::TraversalPath::ROOT
            ? requested.layerStack
            : parentSnapshot.outputFilter.layerStack;
```

A `DisplayDevice` carries its own layer stack (`DisplayDevice.h:132`) and applies it as a `LayerFilter` onto its composition output (`DisplayDevice.cpp:208`). A layer is composited on a display **iff their layer stacks match** (the `LayerFilter::includes` test, Chapter 2.7).

> **This `group_id` match is what the plugin's multi-display behavior is built on.** In the trace, a layer's rect carries a `group_id` = its layer stack, and a display's rect carries a `group_id` = its layer stack. Matching a layer to "its display" is a `group_id == group_id` lookup. The plugin uses exactly this to scope the Surface view to the selected display (Chapter 7).

Physical vs virtual is decided by whether the display has physical panel info:

```cpp
// DisplayDevice.h:304
    bool isVirtual() const { return !physical; }
```

---

## 3.5 Virtual displays and mirroring — how screen recording works

A screen recorder, a `MediaCodec` encoder input surface, a Wi-Fi-Display/Cast sink — none has a real panel. SF models each as a **virtual display**: a `DisplayDevice` with no `physical` info, **its own layer stack**, and an output surface (the sink). The client app calls `createVirtualDisplay`, giving it the sink surface:

```cpp
// SurfaceFlinger.cpp:646
    DisplayDeviceState state;
    state.isSecure = isSecure;
    ...
    state.initialPowerMode = hal::PowerMode::ON;   // Virtual displays start ON.
    state.displayName = displayName;
    mCurrentState.displays.add(token, state);
```

But a virtual display with its own (empty) layer stack would show nothing. To show the screen, the recorder **mirrors** the primary display. Two flavors:

**(a) Display mirroring** — `SurfaceFlinger::mirrorDisplay` creates one "mirror layer" that pulls in another display's *entire* layer stack:

```cpp
// SurfaceFlinger.cpp:5869
    layerStack = display->getLayerStack();
    mirrorArgs.layerStackToMirror = layerStack;
    mirrorArgs.addToRoot = true;
    result = createLayer(mirrorArgs, &outResult.handle, &rootMirrorLayer);
```

**(b) Layer mirroring** — `mirrorLayerHandle` → `layerIdToMirror` at layer creation (`SurfaceFlinger.cpp:5051`).

The key design point — **mirroring clones no layers**:

> "Mirrored layers are represented by the same node in the graph with multiple parents. This allows us to implement mirroring without cloning Layers and maintaining complex hierarchies." — `FrontEnd/readme.md:114`

Because one node has multiple parents, it is reached by multiple **traversal paths**, and **each path produces its own snapshot** — that's how the same content appears on both the phone screen and the recording, possibly with different geometry. A `Detached_Mirror` snapshot even ignores the source's local transform (`LayerSnapshotBuilder.cpp:608`). Layers flagged `eLayerSkipScreenshot` are dropped from mirrored hierarchies (`LayerSnapshotBuilder.cpp:810`) — that's why secure/DRM content vanishes from screenshots and recordings.

> **Why this matters to the plugin.** When you record with the video data source (or just run `screenrecord`), SurfaceFlinger creates that virtual display, and if you're *also* tracing layers, it shows up in `__intrinsic_surfaceflinger_display` with `is_virtual=1` and a mirror layer in its layer stack. The plugin therefore lists it as a (virtual)-tagged display you can select. That single mirror layer is often invisible (it mirrors a layer that exists only to receive touch input, with nothing to draw), which is why selecting that display can show "No visible layers — N hidden" (Chapter 7, Chapter 8). Chapter 6 dissects this overlap.

---

## 3.6 Per-layer geometry and state (the Properties pane, sourced)

Everything the plugin shows in its **Properties** pane comes from these resolved fields. Raw client values live in `layer_state_t`; resolved values land on the `LayerSnapshot`.

### The transform matrix

A layer's transform is a 2×3 affine: a 2×2 linear part plus a translation. The 2×2 part comes from a `matrix22_t`:

```cpp
// gui/LayerState.h:374
struct matrix22_t { float dsdx{0}; float dtdx{0}; float dtdy{0}; float dsdy{0}; };
```

and the translation `(tx, ty)` is the layer's `(x, y)`. Together they map a layer-space point to screen space:

```
| dsdx  dtdx  tx |   | x |   | dsdx·x + dtdx·y + tx |
| dtdy  dsdy  ty | · | y | = | dtdy·x + dsdy·y + ty |
|  0     0    1  |   | 1 |
```

(SF's names aren't the textbook ones, so don't read them by analogy. The **x** output is `dsdx·x + dtdx·y + tx`; the **y** output is `dtdy·x + dsdy·y + ty` — that is what `Transform::transform()` in `libs/ui/Transform.cpp` computes, and what the plugin's `apply()` (Chapter 7.3) reproduces, so a layer lands in the rects view exactly where SF put it on screen.)

The world transform is the parent's transform composed with the local one:
```cpp
// FrontEnd/LayerSnapshotBuilder.cpp:1204
    snapshot.geomLayerTransform = parentSnapshot.geomLayerTransform * snapshot.localTransform;
```

### Bounds and crop (why children are clipped by parents)

```cpp
// FrontEnd/LayerSnapshotBuilder.cpp:1236
    FloatRect parentBounds = parentSnapshot.geomLayerBounds;
    parentBounds = snapshot.localTransform.inverse().transform(parentBounds);
    snapshot.geomLayerBounds = requested.externalTexture ? snapshot.bufferSize.toFloatRect() : parentBounds;
    snapshot.geomLayerCrop = parentBounds;
    if (!requested.crop.isEmpty()) snapshot.geomLayerCrop = snapshot.geomLayerCrop.intersect(requested.crop);
    snapshot.geomLayerBounds = snapshot.geomLayerBounds.intersect(snapshot.geomLayerCrop);
    snapshot.transformedBounds = snapshot.geomLayerTransform.transform(snapshot.geomLayerBounds);
```
So a layer's on-screen rectangle = its bounds (its buffer size or its parent's bounds), intersected with its crop, transformed to the screen. An **invalid transform** (e.g. singular matrix) hides the layer (`LayerSnapshotBuilder.cpp:1206`, `:1228`).

### Color, corner radius, shadow, blur, buffer, input

- **Alpha** multiplies down the tree (`LayerSnapshotBuilder.cpp:818`, `snapshot.color.a = parentSnapshot.color.a * requested.color.a`). A buffered layer's color is just its alpha (`RequestedLayerState.cpp:469`).
- **Corner radius** is a `RoundedCornerState` (`LayerSnapshot.h:33`); the trace carries per-corner `corner_radii` (tl/tr/bl/br).
- **Shadow** (`shadowSettings.length = requested.shadowRadius`) and **blur** (`backgroundBlurRadius`/`blurRegions`) both force GPU composition.
- **Buffer**: active buffer (width/height/stride/format), **dataspace** (e.g. `STANDARD_BT709`), pixel format, frame number — `LayerSnapshot.cpp:501`, `:425`.
- **Input**: `inputInfo` is a `gui::WindowInfo` (`LayerSnapshot.h:103`) with the touchable region, focusable flag, input transform, crop-layer id, and `replaceTouchableRegionWithCrop`. Only populated when input tracing is on. A layer with the `SPY` input config (`1<<14`) is an input "spy" — the plugin's **Spy** chip.
- **Flags** (`gui/LayerState.h:178`): `eLayerHidden=0x01`, `eLayerOpaque=0x02`, `eLayerSkipScreenshot=0x40`, `eLayerSecure=0x80`.

---

## 3.7 Visibility and occlusion

There are **two** stages, and the trace exposes the first.

### Stage 1 — is this layer eligible to be seen? (`is_visible`)

```cpp
// FrontEnd/LayerSnapshot.cpp:225
bool LayerSnapshot::getIsVisible() const {
    if (reachability != Reachability::Reachable) return false;
    ...
    if (!hasSomethingToDraw()) return false;
    if (isHiddenByPolicy()) return false;
    return color.a > 0.0f || hasBlur();
}
```

"Hidden by policy" is the `eLayerHidden` flag **inherited from the parent** (hiding a parent hides its whole subtree) plus an invalid transform (`LayerSnapshot.cpp:221`, `LayerSnapshotBuilder.cpp:762`). SF even produces a human-readable reason string — "hidden by parent or layer flag", "alpha = 0 and no blur", "nothing to draw", "layer only reachable via relative parent" (`LayerSnapshot.cpp:251`). **The importer copies these reason strings into the trace**, and the plugin shows them in the "Invisible due to" curated row (Chapter 5, Chapter 7).

### Stage 2 — occlusion (the composited visible region)

The *region* actually visible after subtracting opaque layers above is computed later, in CompositionEngine:

> "visibleRegion: area of a surface that is visible on screen and not fully transparent. This is essentially the layer's footprint minus the opaque regions above it." — `CompositionEngine/src/Output.cpp:603`

```cpp
// CompositionEngine/src/Output.cpp:649 (sketch)
    if (layerFEState->isOpaque && ...) opaqueRegion.set(geomRect);
    coveredRegion = coverage.aboveCoveredLayers.intersect(visibleRegion);
    coverage.aboveCoveredLayers.orSelf(visibleRegion);
    visibleRegion.subtractSelf(coverage.aboveOpaqueLayers);   // subtract what's covering us
```

The trace_processor importer re-derives a simpler version of this (full-containment and overlap tests against already-seen opaque layers) to produce the **occluded-by / partially-occluded-by / covered-by** lists the plugin shows (Chapter 5.5).

---

## 3.8 What you now know

- A "layer" in the trace is a fully-resolved **`LayerSnapshot`**; the modern FrontEnd owns the tree, the slim legacy `Layer` owns the buffer pipeline.
- The tree is a **graph**; z-order is (stack, z, id); **relative-Z** re-parents a layer for ordering only (`z_order_relative_of`), and unreachable relative parents are flagged (`is_missing_z_parent`).
- A **layer stack** binds layers to a display; matching layer↔display is the `group_id` join the plugin relies on.
- **Virtual displays + mirroring** are how screen recording works, without cloning layers — and why a recorder appears as an extra (virtual) display in the trace.
- Per-layer **geometry** (bounds × crop × transform), color/corner/shadow/blur/buffer/input, and **visibility/occlusion** are exactly the fields the plugin's Surface, Hierarchy, and Properties panes display.

Now we leave the device: how SF serializes this snapshot into a Perfetto trace, and exactly what's in the bytes.

[« Chapter 2](02-surfaceflinger-compositing.md)  ·  [Chapter 4 — The layers trace »](04-the-layers-trace.md)
