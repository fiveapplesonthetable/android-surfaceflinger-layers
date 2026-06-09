# Chapter 1 — Android graphics from zero

*Prerequisite: none. By the end you'll understand how an app's pixels travel to SurfaceFlinger, what double/triple buffering actually is and why it exists, and how vsync and fences keep it all in step without anyone busy-waiting.*

All file citations are under `frameworks/native/` in AOSP branch `android16-qpr2-release`.

---

## 1.1 The problem graphics has to solve

Imagine the naïve design: every app draws directly into the framebuffer (the memory the display scans out). Three things immediately break:

1. **Tearing.** If an app overwrites the framebuffer while the display is half-way through scanning it out, the top of the screen shows the old frame and the bottom shows the new one — a visible horizontal "tear."
2. **No isolation.** Two apps (plus the status bar, plus the wallpaper) would fight over the same memory. There's no way to say "this app owns this rectangle, on top of that one."
3. **Stalls.** The app would have to perfectly time its drawing to the display's scan-out, or block waiting for it, wasting CPU/GPU.

Android's answer has two pillars:

- **Each producer draws into its own off-screen buffer**, never the framebuffer directly. A separate process, **SurfaceFlinger**, owns the final composition.
- **Buffers are handed between producer and consumer through a `BufferQueue`**, with synchronization done by **fences** (so the GPU and display run asynchronously) and timing driven by **vsync** (so frames swap only between refreshes).

The rest of this chapter is those two pillars in detail.

---

## 1.2 Producer and consumer: the two ends of a BufferQueue

The core object is `BufferQueueCore`. It holds all the shared state. It is wrapped by two "faces":

- the **producer** face, `BufferQueueProducer`, implementing the Binder interface `IGraphicBufferProducer` (often abbreviated **IGBP**),
- the **consumer** face, `BufferQueueConsumer`, implementing `IGraphicBufferConsumer` (**IGBC**).

They can live in different processes (the interfaces are Binder interfaces), which is the normal case: producer in the app, consumer in SurfaceFlinger. The factory builds all three and hands back only the two interfaces:

```cpp
// libs/gui/BufferQueue.cpp:118
void BufferQueue::createBufferQueue(sp<IGraphicBufferProducer>* outProducer,
        sp<IGraphicBufferConsumer>* outConsumer,
        bool consumerIsSurfaceFlinger) {
    sp<BufferQueueCore> core = sp<BufferQueueCore>::make();
    sp<IGraphicBufferProducer> producer = sp<BufferQueueProducer>::make(core, consumerIsSurfaceFlinger);
    sp<IGraphicBufferConsumer> consumer = sp<BufferQueueConsumer>::make(core);
    *outProducer = producer;
    *outConsumer = consumer;
}
```

The two faces are `friend`s of the core so they can touch its private state directly (`BufferQueueCore.h:58`).

### What an app's "Surface" is

When an app draws — with Canvas, OpenGL ES, or Vulkan — it draws into a **`Surface`**. A `Surface` is the producer side, dressed up as the C `ANativeWindow` API that EGL/Vulkan/Canvas know how to target:

```cpp
// libs/gui/include/gui/Surface.h:107
class Surface : public ANativeObjectBase<ANativeWindow, Surface, RefBase>
...
    sp<IGraphicBufferProducer> mGraphicBufferProducer;   // :546 — the producer it wraps
```

So: **"Surface" (the producer) = an `ANativeWindow` backed by an `IGraphicBufferProducer`.** Keep this distinct from a SurfaceFlinger **Layer** (the consumer-side object, Chapter 3). The naming collision is one of Android graphics' cruelest beginner traps.

### The slot array — why buffers aren't copied or re-sent

The clever part: the actual pixel buffers are **not** sent over Binder every frame, and never copied. Instead both ends keep a **slot table** — up to 64 entries — and per frame they exchange only a small integer (the slot index) plus a fence. A `GraphicBuffer` is transferred over Binder exactly once, the first time a slot is filled (`requestBuffer`). The header says it outright:

```cpp
// libs/gui/include/gui/IGraphicBufferProducer.h:111
    // requestBuffer requests a new buffer for the given index. The server ...
    // assigns the newly created buffer to the given slot index, and the client
    // is expected to mirror the slot->buffer mapping so that it's not necessary
    // to transfer a GraphicBuffer for every dequeue operation.
```

The slot count ceiling:
```cpp
// libs/ui/include/ui/BufferQueueDefs.h:25
static constexpr int NUM_BUFFER_SLOTS = 64;
```

The core owns the canonical slot table, a FIFO of queued buffers, and free-sets:
```cpp
// libs/gui/include/gui/BufferQueueCore.h:215, :222
    BufferQueueDefs::SlotsType mSlots;     // the slot table (mirrored on both sides)
    Fifo mQueue;                           // queued buffers waiting for the consumer
    std::set<int> mFreeSlots;              // FREE, no buffer attached
    std::list<int> mFreeBuffers;           // FREE, buffer still attached (reusable)
    std::set<int> mActiveBuffers;          // non-FREE
```

### A slot's state: FREE → DEQUEUED → QUEUED → ACQUIRED

Each slot has a `BufferState`. Conceptually it's a four-state machine (plus a SHARED special case). The canonical table is documented right in the struct:

```
// libs/gui/include/gui/BufferSlot.h:50
         | mShared | mDequeueCount | mQueueCount | mAcquireCount |
FREE     |  false  |       0       |      0      |       0       |
DEQUEUED |  false  |       1       |      0      |       0       |
QUEUED   |  false  |       0       |      1      |       0       |
ACQUIRED |  false  |       0       |      0      |       1       |
SHARED   |  true   |      any      |     any     |      any      |
```

with ownership documented at `BufferSlot.h:60`:
- **FREE** — owned by the BufferQueue, available to be dequeued by the producer.
- **DEQUEUED** — owned by the producer, which "may modify the buffer's contents as soon as the associated release fence is signaled."
- **QUEUED** — filled by the producer, waiting for the consumer; "contents must not be accessed until the associated fence is signaled."
- **ACQUIRED** — owned by the consumer; returns to FREE when `releaseBuffer` is called.

(Implementation note for accuracy: the state is stored as three counters, not a single enum — see the transitions at `BufferSlot.h:115`. The four names are conceptual.)

---

## 1.3 The life of one frame's buffer

Here is the full cycle. Notice that pixels move by *reference* (a slot index) and synchronization is by *fence*, never by blocking on the GPU.

```
            dequeueBuffer          queueBuffer            acquireBuffer
   ┌──► FREE ───────────► DEQUEUED ───────────► QUEUED ───────────────► ACQUIRED ──┐
   │   (BufferQueue       (producer/app          (in the FIFO,           (consumer/  │
   │    owns it)           draws into it)         + acquire fence)        SF reads)   │
   │                                                  │                               │
   │                                          async mode: newest                      │
   │                                          frame overwrites the                    │
   │                                          old queued one (drop)         releaseBuffer
   └──────────────────────────────────────────────────────────────────────  ◄───────┘
        notifyBufferReleased() wakes a producer blocked in dequeueBuffer    (+ release fence)
```


### Step 1 — `dequeueBuffer`: the producer takes a FREE slot

```cpp
// libs/gui/BufferQueueProducer.cpp:449 (method), :577 (the transition)
mSlots[found].mBufferState.dequeue();   // FREE -> DEQUEUED
```
If the slot has no buffer, or the requested size/format changed, the producer allocates a fresh `GraphicBuffer` for it (`:609`). Crucially, `dequeueBuffer` hands back the slot's **release fence** (`:631`):
```cpp
    *outFence = mSlots[found].mFence;   // the RELEASE fence from the previous consumer
    mSlots[found].mFence = Fence::NO_FENCE;
```
This fence signals when the *previous consumer* finished reading this buffer. The producer must wait on it (on the GPU, asynchronously) before it overwrites the buffer. It does **not** CPU-block here.

### Step 2 — the app renders

The app draws into the dequeued buffer via the `ANativeWindow`. The GPU work is asynchronous; its completion is captured by an **acquire fence**, which the app will hand over in the next step.

### Step 3 — `queueBuffer`: publish to QUEUED

```cpp
// libs/gui/BufferQueueProducer.cpp:1059
        mSlots[slot].mFence = acquireFence;       // store the acquire fence on the slot
        mSlots[slot].mBufferState.queue();        // DEQUEUED -> QUEUED
        ++mCore->mFrameCounter;
```
The queued buffer (with its acquire fence, target present time, crop, transform) is appended to the FIFO, and the consumer's `onFrameAvailable` listener is notified (`:1110`). The acquire fence rides along inside the `BufferItem` so the consumer knows when the pixels are actually ready.

### Step 4 — `acquireBuffer`: the consumer takes the front → ACQUIRED

```cpp
// libs/gui/BufferQueueConsumer.cpp:280, :300
        mSlots[slot].mBufferState.acquire();   // QUEUED -> ACQUIRED
        ...
        mCore->mQueue.erase(front);            // removed from the FIFO
```
The consumer gets the acquire fence in the returned `BufferItem` and must wait on it before *reading* the pixels.

### Step 5 — `releaseBuffer`: back to FREE

```cpp
// libs/gui/BufferQueueConsumer.cpp:526, :538, :546
        mSlots[slot].mFence = releaseFence;       // store the RELEASE fence
        mSlots[slot].mBufferState.release();      // ACQUIRED -> FREE
        ...
        mCore->mFreeBuffers.push_back(slot);      // reusable
        mCore->notifyBufferReleased();            // wake any producer blocked in dequeueBuffer
```
That `notifyBufferReleased()` is what unblocks a producer that ran out of buffers. The circle is closed: the freed slot can be dequeued again.

---

## 1.4 Double vs triple buffering — finally, precisely

Now the headline concept, which most explanations get vague about. In this codebase **the buffer count is not a hard-coded "2" or "3."** It is *derived* from three knobs.

### The three knobs

```cpp
// libs/gui/BufferQueueCore.cpp:123 (defaults)
        mMaxBufferCount(BufferQueueDefs::NUM_BUFFER_SLOTS),  // 64 = absolute ceiling
        mMaxAcquiredBufferCount(1),   // how many the CONSUMER may hold at once
        mMaxDequeuedBufferCount(1),   // how many the PRODUCER may hold at once
```

### The formula

```cpp
// libs/gui/BufferQueueCore.cpp:263
int BufferQueueCore::getMaxBufferCountLocked() const {
    int maxBufferCount = mMaxAcquiredBufferCount + mMaxDequeuedBufferCount +
            ((mAsyncMode || mDequeueBufferCannotBlock) ? 1 : 0);
    return std::min(mMaxBufferCount, maxBufferCount);
}
```

Read it off directly:

- **Defaults, blocking mode:** `1 (acquired) + 1 (dequeued) + 0 = 2` → **double buffering.** The producer holds one to draw into; the consumer holds one to show. They swap.
- **Async / non-blocking dequeue:** `1 + 1 + 1 = 3` → **triple buffering.** The extra `+1` is the whole point of triple buffering.
- **High refresh / deep pipeline** (consumer allowed 2): `2 + 1 + 1 = 4`.

### Why the "+1" — the non-blocking guarantee

```cpp
// libs/gui/BufferQueueCore.cpp:241
int BufferQueueCore::getMinUndequeuedBufferCountLocked() const {
    if (mAsyncMode || mDequeueBufferCannotBlock) {
        return mMaxAcquiredBufferCount + 1;   // the extra buffer
    }
    return mMaxAcquiredBufferCount;
}
```
This number is reported to the app (via `NATIVE_WINDOW_MIN_UNDEQUEUED_BUFFERS`) so EGL sizes its swap chain correctly. The "+1" guarantees the producer always has **at least one buffer it can dequeue without blocking**. That spare buffer is *exactly* what lets the app start frame N+2 while frame N+1 (finished) waits to be shown and frame N is on screen.

### The tradeoff, concretely

- **Double buffering (2):** lowest latency (a finished frame is shown at the very next vsync), but if the app is still drawing when vsync arrives, there is *no fresh buffer* and the previous frame is shown again — **a dropped frame / jank.** The producer must often **block** in `dequeueBuffer` waiting for a release:
  ```cpp
  // libs/gui/BufferQueueProducer.cpp:435
  status_t status = waitForBufferRelease(lock, mDequeueTimeout);   // blocks on a condvar
  ```
- **Triple buffering (3):** the producer almost never blocks (there's always a spare), smoothing throughput and hiding occasional long frames — at the cost of **one extra frame of latency** (a frame may sit finished-but-unshown for a refresh).

AOSP even ships a design note on the failure mode, which is a perfect teaching example:

> "Buffer stuffing happens on the client side when SurfaceFlinger misses a frame, but the client continues producing buffers at the same rate. ... SurfaceFlinger cannot yet release the buffer for the frame that it missed and the client has one less buffer to render into. The client may then run out of buffers ... increasing the chances of janking."
> — `libs/gui/BufferStuffing.md`

### Async mode = "newest frame wins"

There's a related mode where `queueBuffer` never blocks: instead of waiting, it **drops** the previously-queued (un-consumed) frame and overwrites it:

```cpp
// libs/gui/BufferQueueProducer.cpp:1116 (the replace)
            if (last.mIsDroppable) {
                mSlots[last.mSlot].mBufferState.freeQueued();   // QUEUED -> FREE (dropped)
                mCore->mQueue.editItemAt(mCore->mQueue.size() - 1) = item;  // overwrite newest
            }
```
This is why a fast game keeps the queue one deep and always shows the freshest frame, discarding stale ones rather than building latency.

### Who picks the consumer's acquire count?

The consumer's `mMaxAcquiredBufferCount` (default 1) is computed by SurfaceFlinger from the refresh rate and pipeline depth — deeper pipelines (higher Hz) acquire more, hence allocate more buffers:

```cpp
// services/surfaceflinger/SurfaceFlinger.cpp:8889
int SurfaceFlinger::calculateMaxAcquiredBufferCount(Fps refreshRate,
                                                    std::chrono::nanoseconds presentLatency) {
    int64_t pipelineDepth = presentLatency.count() / refreshRate.getPeriodNsecs();
    ...
    return std::max(minAcquiredBuffers, maxAcquiredBuffers);
}
```

---

## 1.5 vsync, latching, and the three fences

### vsync

The display refreshes at fixed instants — vsync. SurfaceFlinger does not poll; it is woken once per vsync by the scheduler (the HAL vsync entry is `SurfaceFlinger.cpp:2452`, `onComposerHalVsync`). Apps schedule their drawing one vsync ahead of the present they're targeting via **Choreographer** (`libs/gui/Choreographer.cpp`), which is fed by a `DisplayEventReceiver`/`DisplayEventDispatcher`. The full SF wake→commit→composite loop is Chapter 2; here we care about one step of it: **latching**.

### Latching = choosing which queued buffer to show this frame

Each vsync, for each layer with a ready frame, SF "latches" a buffer — binds the chosen queued buffer as the layer's current content for this frame. The latch is gated on the buffer's **acquire fence**: SF will not latch (and not composite) until the GPU has actually finished drawing it.

```cpp
// services/surfaceflinger/Layer.cpp:1475
bool Layer::latchBufferImpl(bool& recomputeVisibleRegions, nsecs_t latchTime,
                            nsecs_t expectedPresentTime, bool bgColorOnly) {
    ...
    // If the head buffer's acquire fence hasn't signaled yet, return and
    // try again later
    if (!fenceHasSignaled()) {
        mFlinger->onLayerUpdate();   // ask for another wakeup
        return false;
    }
    updateTexImage(latchTime, expectedPresentTime, bgColorOnly);   // commit it as "drawing"
```

The BufferQueue itself also does vsync-aware timing inside `acquireBuffer`: given the expected present time it will **drop** queued frames that are already overdue (so a newer due frame shows) and **defer** a frame whose target present time is still in the future (returning `PRESENT_LATER`) — `BufferQueueConsumer.cpp:149` and `:214`.

### The three fences (the heart of asynchrony)

| Fence | Direction | Means |
|---|---|---|
| **acquire fence** | producer → consumer | "the GPU finished *writing* this buffer." SF must wait before compositing. |
| **release fence** | consumer → producer | "compositing/scan-out finished *reading* this buffer." The producer must wait before overwriting. |
| **present fence** | display (HWC) → SF | "this frame actually *hit the screen* at time T." |

The acquire fence is created by the app and travels in `queueBuffer` → `BufferItem` → SF's latch. The release fence is created by SF/HWC and travels back in `releaseBuffer` → the slot → the producer's next `dequeueBuffer`. The present fence comes from the composer HAL after present, and SF feeds it to the vsync predictor and frame-timing telemetry:

```cpp
// services/surfaceflinger/SurfaceFlinger.cpp:3560, :3778
    auto presentFence = getHwComposer().getPresentFence(id);
    ...
    mScheduler->addPresentFence(id, std::move(presentFence));   // drives vsync prediction
```

Because everything is a fence, the CPU thread that runs SF never blocks on the GPU: it queues work, attaches fences, and moves on; the hardware signals fences when it's actually done.

---

## 1.6 BLAST: how an app's frames reach SF today

One modernization is worth knowing, because it explains why "the app's BufferQueue consumer" isn't literally inside SurfaceFlinger anymore. Each window has a **BLASTBufferQueue (BBQ)** living *in the app process*. The app's `Surface` is the producer; **BBQ holds the consumer**. When the app queues a frame, BBQ acquires it and turns it into a **SurfaceFlinger transaction** carrying the buffer (`setBuffer`), applied to the window's `SurfaceControl`:

```cpp
// libs/gui/BLASTBufferQueue.cpp:589, :656
    status_t status = mBufferItemConsumer->acquireBuffer(&bufferItem, 0 /*expectedPresent*/, false);
    ...
    t->setBuffer(mSurfaceControl, buffer, fence, bufferItem.mFrameNumber, mProducerId,
                 releaseBufferCallback, dequeueTime);
```

So a frame reaches the screen in two hops:

1. **per queued frame:** app `queueBuffer` → BBQ `onFrameAvailable` → `acquireBuffer` → `setBuffer` transaction (in the app process).
2. **per vsync:** SF applies the transaction in **commit**, latches the buffer (gated on the acquire fence), composites in **composite**, presents; then returns a **release fence** through the transaction-completed callback, which BBQ uses to `releaseBuffer` back into the BufferQueue, freeing the producer's slot.

This is why Chapter 2 talks about "transactions carrying buffers" — that's BBQ at work. It does not change anything in this chapter conceptually: it's still producer→BufferQueue→consumer, fences and all; the consumer just happens to be a per-window object that forwards via transactions.

---

## 1.7 What you now know

- An app draws into its **own buffer**, dequeued from a **BufferQueue**, never the framebuffer.
- Buffers move by **slot index + fence**, not by copying or re-sending.
- A buffer cycles **FREE → DEQUEUED → QUEUED → ACQUIRED → FREE**.
- **Double vs triple buffering** is `acquired + dequeued + (async ? 1 : 0)`; the extra buffer trades a frame of latency for non-blocking throughput (less jank).
- **vsync** wakes SF; **latching** picks each layer's buffer for the frame, gated on its **acquire fence**; **release** and **present** fences close the loop without busy-waiting.
- Today an app's frames reach SF via **BLAST** transactions, but the model underneath is unchanged.

Everything so far has been about *one* layer's buffer getting to SurfaceFlinger. Next: what SF *does* with all the layers each vsync — the commit/composite loop, and how it decides between hardware overlays and the GPU.

[« README](README.md)  ·  [Chapter 2 — How SurfaceFlinger composites a frame »](02-surfaceflinger-compositing.md)
