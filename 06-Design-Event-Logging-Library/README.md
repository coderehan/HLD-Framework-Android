# 📊 HLD — Design Event Logging Library (Android SDK)

**Examples:** Firebase Analytics SDK, Segment, Mixpanel SDK, Amplitude SDK

> Topic #6 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.
>
> ⚠️ **Note:** This is a **library/SDK**, not a client-server app — it lives *inside* the host app and talks to a backend, but has no UI of its own. The framework still applies, just the "diagram" is internal SDK components instead of microservices.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the (developer/host app) DO with this library?**

| # | Requirement |
|---|---|
| 1 | App can log an event with a name + properties (e.g. `"item_purchased"`, `{price: 499}`) |
| 2 | Library can batch multiple events together before sending (efficiency) |
| 3 | Library can persist unsent events to disk (survive app kill/crash) |
| 4 | Library can upload events to a backend server |
| 5 | Library can retry failed uploads with backoff |
| 6 | Library can attach common metadata automatically (user ID, device info, timestamp, session ID) |

> **Rule of thumb:** the "user" here is another developer calling `Logger.log(event)` — think API design, not screens.

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| 🔋 **Battery Efficiency** | Must NOT wake the network radio for every single event — this is the #1 complaint about bad logging SDKs |
| 📴 **Reliability (no data loss)** | Events shouldn't vanish if the app crashes or is force-killed before upload |
| ⚡ **Low Overhead** | `log()` call must return instantly — never block the caller's main thread |
| 📈 **Scalability** | Must handle a chatty app firing hundreds of events per session without choking |
| 🔧 **Maintainability** | Easy to plug in a new backend/destination without changing app code |

> Skipped **Low Latency** on the *delivery* side — events don't need to reach the server in real-time, only reliably, eventually.

---

## 🎨 Step 3: HLD Diagram

### Base structure (adapted — no "User → UI" here, it's "Host App → SDK")
```
              Host App
                 |
                 ↓
         Logger.log(event)   ← public API, returns instantly
                 |
                 ↓
           Event Queue (in-memory)
                 |
                 ↓
         Batching Processor
                 |
                 ↓
        Local Persistence (disk)
                 |
                 ↓
          Upload Manager
           (WorkManager)
                 |
                 ↓
             Backend
```

### Extended internal flow
```
┌──────────┐  1. log(event)   ┌─────────────────┐
│ Host App │ ───────────────▶ │ Public API        │
└──────────┘                  │ (Logger.log)       │
                               └────────┬────────┘
                                        │ 2. enqueue (non-blocking)
                                        ▼
                               ┌─────────────────┐
                               │ In-Memory Queue    │
                               └────────┬────────┘
                                        │ 3. batch when full/timeout
                                        ▼
                               ┌─────────────────┐
                               │ Batching Processor  │
                               │ (e.g. every 20        │
                               │  events or 30 sec)      │
                               └────────┬────────┘
                                        │ 4. write to disk
                                        ▼
                               ┌─────────────────┐
                               │ Local DB (Room/    │
                               │ file-based buffer)  │
                               └────────┬────────┘
                                        │ 5. scheduled upload
                                        ▼
                               ┌─────────────────┐
                               │ Upload Manager      │
                               │ (WorkManager, retries│
                               │  with backoff)        │
                               └────────┬────────┘
                                        │ 6. HTTP POST batch
                                        ▼
                               ┌─────────────────┐
                               │ Backend Ingestion   │
                               │ Endpoint              │
                               └─────────────────┘
                                        │ 7. on success
                                        ▼
                               Delete uploaded events
                               from Local DB
```

**When you actually draw this in Excalidraw:** this is a straight pipeline (not branching flows like the earlier apps) — draw it top-to-bottom as ONE diagram, numbering each stage 1-7. Simpler than the client-server topics.

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **Public API (`Logger.log()`)** | Simple entry point for the host app; must be thread-safe and instant |
| **In-Memory Queue** | Temporarily holds events before batching — avoids disk I/O on every single call |
| **Batching Processor** | Groups events by count or time threshold, whichever comes first |
| **Local DB (persistence)** | Durable buffer — survives app kill/crash so nothing is lost before upload |
| **Upload Manager (WorkManager)** | Schedules uploads respecting battery/network constraints, retries on failure |
| **Backend Ingestion Endpoint** | Receives the batch, stores it for later analytics processing |

### B. Data flow — walking through "Logging an Event"
```
1. Host app calls Logger.log("item_purchased", {price: 499})
2. SDK auto-attaches metadata (timestamp, user ID, session ID, device info)
3. Event is pushed into the in-memory queue — log() call returns immediately
4. Batching Processor waits until either:
   - 20 events have queued up, OR
   - 30 seconds have passed
   whichever happens first — then flushes the batch
5. Batch is written to Local DB (so it survives even if the app is killed now)
6. Upload Manager (WorkManager) picks it up when network is available
   and battery isn't constrained, POSTs it to the backend
7. On success, those events are deleted from Local DB
   On failure, WorkManager retries with exponential backoff
```

### C. Why these choices (tie back to NFRs)
- **In-memory queue + batching** exists because of **Battery Efficiency** — firing a network call per event would constantly wake the radio; grouping into batches means far fewer, larger network calls.
- **Local DB persistence before upload** exists because of **Reliability** — if the app crashes right after `log()`, the event is already safely on disk, not just sitting in memory waiting to be lost.
- **WorkManager for uploads** exists because of **Battery Efficiency + Scalability** — it respects system constraints (only upload on WiFi, only when charging, etc. if configured) instead of the SDK managing its own background thread naively.
- **`log()` returns instantly (non-blocking)** exists because of **Low Overhead** — the host app's UI thread must never stall because of a logging call.

### D. One trade-off to mention
> "Batching by count-or-time means events aren't uploaded the instant they happen — there's a small delay (up to 30 sec). That's an acceptable trade-off here because analytics events don't need real-time delivery, but it wouldn't be OK for something like a chat message."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **What if the local DB fills up (app offline for days)?** | Cap max stored events (e.g. 10k) or max disk size, drop oldest when full |
| **Preventing duplicate uploads on retry** | Assign each batch a unique ID; backend dedupes by that ID |
| **Supporting multiple backends (e.g. also send to Firebase)** | Plugin/destination pattern — Upload Manager loops over registered destinations |
| **High-priority events (e.g. crash logs)** | Separate high-priority queue that bypasses batching and uploads immediately |

---

## ✅ Recap in One Line
> log() → queue (fast, non-blocking) → batch (fewer network calls) → persist to disk (never lose data) → upload via WorkManager (battery-respecting, retries safely).

---
