# Chapter 10 — Transactions, and the other Winscope viewers

*Prerequisite: Chapters 4–5. The layers source is one of a **family** of Winscope data sources that all flow through the same trace_processor importer and all benefit from the same trace-bounds fix. This chapter covers the **transactions** source (the *input* side of the layer tree) and situates the rest: WindowManager, IME, ViewCapture, ProtoLog, Shell Transitions. It's why the bounds fix says "winscope tables," not "SurfaceFlinger tables."*

---

## 10.1 Transactions: the input side of the layer tree

Chapter 3 showed the layer tree as a *result*. But clients never mutate that tree directly — they submit **transactions** (`setLayer`, `setPosition`, `setBuffer`, `setCrop`, `setAlpha`, reparent, …), which SurfaceFlinger commits atomically per vsync (Chapter 2.2). The relationship:

- **`android.surfaceflinger.transactions`** records the *stream of edits* (the deltas).
- **`android.surfaceflinger.layers`** records *snapshots of the materialized tree* (the results).

Transactions are the cause; a layers snapshot is the effect of applying all causes up to that instant.

### Config: two modes

```proto
// external/perfetto/protos/perfetto/config/android/surfaceflinger_transactions_config.proto:23
message SurfaceFlingerTransactionsConfig {
  enum Mode {
    MODE_UNSPECIFIED = 0;
    MODE_CONTINUOUS = 1;  // dump SF's internal transaction ring buffer on each flush
    MODE_ACTIVE = 2;      // write the initial state, then every incoming transaction
  }
  optional Mode mode = 1;
}
```
(Data source name `android.surfaceflinger.transactions`, `TransactionDataSource.h:62`.)

### Schema: an entry is one committed batch

Each `TracePacket` carries a `TransactionTraceEntry` in **field 94** (right next to layers' field 93):
```proto
// external/perfetto/protos/perfetto/trace/trace_packet.proto:232
    TransactionTraceEntry surfaceflinger_transactions = 94;
```
```proto
// surfaceflinger_transactions.proto:51
message TransactionTraceEntry {
  optional int64 elapsed_realtime_nanos = 1;
  optional int64 vsync_id = 2;
  repeated TransactionState transactions = 3;
  repeated LayerCreationArgs added_layers = 4;
  repeated uint32 destroyed_layers = 5;
  repeated DisplayState added_displays = 6;
  ...
}
```
The interesting nested message is `LayerState` (`surfaceflinger_transactions.proto:118`, commented "Keep insync with layer_state_t") — it mirrors SF's native `layer_state_t` and carries a 64-bit **changed-fields bitmask** (`what`) split into `ChangesLsb`/`ChangesMsb` enums (`ePositionChanged`, `eLayerChanged`, `eAlphaChanged`, `eBufferChanged`, `eReparent`, …) plus the new values (`x/y/z`, `layer_stack`, `matrix`, `alpha`, `color`, `crop`, `buffer_data`, `transform`, `corner_radii`, …). This is literally the wire form of the setters from Chapter 2.2. `LayerCreationArgs` (`:79`) describes a newly-created layer (`name`, `parent_id`, `mirror_from_id`, `layer_stack_to_mirror` — the mirroring of Chapter 3.5).

### How it connects to layers: MODE_GENERATED replay

The layers config has a `MODE_GENERATED` (Chapter 4.5) that **reconstructs layer snapshots from the transaction stream** — it replays the transaction ring buffer through SF's front-end logic. The engine is `LayerTraceGenerator`:

> "Generates layer traces from transaction traces… The transactions are parsed from proto and applied to recreate the layer state. The result is then written as a layer trace." — `Tracing/tools/readme.md:3`

So transactions are the source of truth, and layers snapshots can be *derived* from them. That's the deep link between the two sources.

### trace_processor tables

```python
# winscope_tables.py
__intrinsic_surfaceflinger_transactions     # one row per committed batch: ts, arg_set_id,
                                            #   base64_proto_id, vsync_id
__intrinsic_surfaceflinger_transaction      # one row per individual effect: snapshot_id,
                                            #   arg_set_id, base64_proto_id, transaction_id,
                                            #   pid, uid, layer_id, display_id,
                                            #   flags_id, transaction_type
__intrinsic_surfaceflinger_transaction_flag # decoded `what` bits: flags_id, flag (string)
```
`transaction_type` is a parser-assigned string (`LAYER_CHANGED`, `LAYER_ADDED`, `DISPLAY_CHANGED`, `LAYER_DESTROYED`, …), and the importer decodes the `what` bitmask into human-readable flag rows. (Parser: `surfaceflinger_transactions_parser.cc`.)

> The `com.android.SurfaceFlinger` plugin does **not** render transactions today — it's the layers viewer. But the transactions source is captured in the demo config's sibling, and because its `ts` feeds the bounds fix (§10.3), a transactions-only trace also opens. A transactions UI would be a natural next plugin.

---

## 10.2 The other Winscope domains (one importer, many viewers)

Winscope isn't only SurfaceFlinger. The same trace_processor module parses six domains; here's each, with its data source, its `TracePacket` route, and its table(s). They're listed so you understand the shape the SF viewer fits into — and what future plugins could light up (the tables already exist).

| Domain | Data source | Packet route | Captures | Tables |
|---|---|---|---|---|
| **SF layers** | `android.surfaceflinger.layers` | field 93 | the layer tree (this book) | `surfaceflinger_layers_snapshot/_layer/_display` + rects |
| **SF transactions** | `android.surfaceflinger.transactions` | field 94 | the edit stream (§10.1) | `surfaceflinger_transactions/_transaction/_transaction_flag` |
| **WindowManager** | `android.windowmanager` | extensions §6 | the WM hierarchy: displays, tasks, activities, windows, focus, bounds | `windowmanager`, `windowmanager_windowcontainer` |
| **IME** | `android.inputmethod` | extensions §1/2/3 | InputMethod state from client / IME-service / system-manager vantage points | `inputmethod_clients`, `inputmethod_manager_service`, `inputmethod_service` |
| **ViewCapture** | `android.viewcapture` | extensions §4 | the app-process **View** tree (ids, classes, visibility, geometry) — the per-app companion to SF layers | `viewcapture`, `viewcapture_view`, `viewcapture_interned_data` |
| **ProtoLog** | `android.protolog` | fields 104/105 | structured system log messages (level/tag/message/location), de-interned via a viewer-config dictionary | `protolog` |
| **Shell Transitions** | `com.android.wm.shell.transition` | fields 96/97 | WM-Shell animated-transition lifecycle (create/send/merge/finish times, type, handler, SF-transaction ids, participants) | `window_manager_shell_transitions` (+ handlers/participants/protos) |

Two structural notes:

- **One module.** All of these are parsed by a single `WinscopeModule` (`winscope_module.cc`), which registers for the top-level fields (93, 94, 96, 97, 104, 105) and a `WinscopeExtensions` field (112) that *packs* the newer domains (IME, ViewCapture, WindowManager, input events) as **numbered sub-fields** — the "extensions §N" in the table above — demuxed in `ParseWinscopeExtensionsData`. So adding a domain didn't require a new top-level packet field each time.
- **Same pattern as SF layers.** Each domain's *snapshot* table has the same `ts` / `arg_set_id` / `base64_proto_id` triple you learned in Chapter 5 — a timestamp, the flattened proto as args, and the interned raw bytes — so everything you know about reading the SF layers tables transfers directly. (The one exception is **ProtoLog**, whose `__intrinsic_protolog` is a per-*message* table with typed columns `ts, level, tag, message, stacktrace, location` instead of the args/base64 pair — it's log lines, not proto snapshots.)

### Why "ViewCapture" matters for the SF plugin's limitations

Chapter 8 listed "color-by-content, touch pointers, input rays" as gaps because a layers-only trace doesn't carry that data. ViewCapture is *where some of it lives* — the app's actual view tree and content. A future plugin (or an extension of this one) that joins SF layers to ViewCapture by window could fill those gaps. The data path already exists in trace_processor; only the UI is missing.

---

## 10.3 One bounds fix covers them all

This is why Chapter 5.6's fix iterates **nine** tables, not one. The winscope importer's `GetTimestampBounds` min/max-folds `ts` across every domain's snapshot table:

```cpp
// winscope_importer.cc:63
include(s.surfaceflinger_layers_snapshot_table().IterateRows());     // SF layers
include(s.surfaceflinger_transactions_table().IterateRows());        // SF transactions (§10.1)
include(s.windowmanager_table().IterateRows());                      // WindowManager
include(s.window_manager_shell_transitions_table().IterateRows());   // Shell transitions
include(s.inputmethod_clients_table().IterateRows());                // IME (client)
include(s.inputmethod_manager_service_table().IterateRows());        // IME (manager)
include(s.inputmethod_service_table().IterateRows());                // IME (service)
include(s.viewcapture_table().IterateRows());                        // ViewCapture
include(s.protolog_table().IterateRows());                           // ProtoLog
```
and `GetBoundsMutationCount` enumerates the same nine (the parity the comment requires, Chapter 5.6). So the fix isn't "make the SF layers viewer open" — it's "**make any winscope-only trace open**." Capture only WindowManager, or only ProtoLog, and the timeline now has valid bounds.

(Caveat: there isn't a dedicated input-event snapshot table in this set, so a trace containing *only* `android_input_event` and nothing else might still have empty bounds — check that case if input-only traces matter to you. And, as of writing, this importer-plugin layout plus the fix live in upstream Perfetto, not yet in AOSP's vendored `external/perfetto`.)

---

## 10.4 Summary

- **Transactions** are the input deltas; **layers** are the materialized result; `MODE_GENERATED` can rebuild the latter from the former.
- The SF layers viewer is one member of a family — WM, IME, ViewCapture, ProtoLog, Shell Transitions — all parsed by **one** trace_processor importer, all with the same `ts`/`arg_set_id`/`base64_proto_id` table shape.
- The trace-bounds fix is deliberately for the **whole family**, so any winscope-only trace opens — and the other domains' tables are already there for future plugins to render.

[« Chapter 9](09-one-layer-end-to-end.md)  ·  [Chapter 11 — Capture it yourself »](11-capture-and-explore.md)
