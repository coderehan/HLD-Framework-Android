# 📤 HLD — Design Image Upload Library (Android)

**Examples:** Uploadcare SDK, AWS S3 Transfer Utility, Firebase Storage SDK

> Topic #9 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.
>
> ⚠️ **Note:** Yet another SDK shape. Event Logging = "ship small data reliably." Networking SDK = "make one request cleanly." This one = **"move a large file reliably over an unreliable connection, and let the user watch it happen."** The big new ideas here: **compression before upload**, **progress reporting**, and **resumable/chunked uploads**.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the (developer/host app) DO with this library?**

| # | Requirement |
|---|---|
| 1 | App can upload an image given a local file/URI |
| 2 | App can observe upload progress (0-100%) |
| 3 | Library can compress/resize the image before uploading |
| 4 | Library can resume an interrupted upload instead of restarting from zero |
| 5 | App can cancel an in-progress upload |
| 6 | Library can retry automatically on failure (with backoff) |
| 7 | App can queue multiple uploads (e.g. uploading a whole photo album) |

> **Rule of thumb:** compare this to the Networking SDK topic — that one assumed small JSON payloads that finish in milliseconds. Here, a single file can take seconds to minutes, over spotty mobile networks. That difference drives almost every design decision below.

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| 🔒 **Reliability** | A half-uploaded file on a dropped connection shouldn't mean starting over |
| 🔋 **Battery/Data Efficiency** | Compress before sending — don't waste data/battery uploading a 10MB photo when 500KB will do |
| ⚡ **Responsiveness** | Progress updates must feel smooth (no jumping from 10% to 100%) |
| 📴 **Resilience to poor network** | Mobile networks drop mid-upload constantly — must handle this gracefully |
| 🔧 **Concurrency Control** | Multiple queued uploads shouldn't all fire at once and saturate the connection |

> Skipped strict **Consistency** — unlike an order or a trade, a slightly-delayed or retried image upload has no correctness risk, just a UX cost.

---

## 🎨 Step 3: HLD Diagram

### Base structure (SDK-style — pipeline, not client-server)
```
         App calls upload(fileUri)
                    |
                    ↓
             Upload Manager
             (queue + concurrency limit)
                    |
                    ↓
           Compression Stage
           (resize/compress image)
                    |
                    ↓
             Chunking Stage
           (split into parts, e.g. 1MB each)
                    |
                    ↓
           Chunk Upload Executor
          (uploads chunks, tracks which
           succeeded, retries failed ones)
                    |
                    ↓
                 Server
                    |
                    ↓
           Progress Callback
           (back to the App's UI)
```

### Extended internal flow
```
┌──────────┐ 1. upload(fileUri) ┌─────────────────┐
│ App      │ ──────────────────▶│ Upload Manager    │
└──────────┘                    │ (adds to queue)     │
                                 └────────┬────────┘
                                          │ 2. dequeue when a
                                          │    concurrency slot frees up
                                          ▼    (e.g. max 2 uploads at once)
                                 ┌─────────────────┐
                                 │ Compression Stage  │
                                 │ (resize + compress) │
                                 └────────┬────────┘
                                          │ 3. compressed file
                                          ▼
                                 ┌─────────────────┐
                                 │ Chunking Stage      │
                                 │ splits into N parts   │
                                 └────────┬────────┘
                                          │ 4. upload chunk-by-chunk
                                          ▼
                                 ┌─────────────────┐
                                 │ Chunk Upload         │
                                 │ Executor               │
                                 │ (tracks chunk state:   │
                                 │  PENDING/DONE/FAILED)  │
                                 └────────┬────────┘
                                          │ 5. PUT each chunk
                                          ▼
                                 ┌─────────────────┐
                                 │ Server (Upload       │
                                 │ Ingestion Endpoint)    │
                                 └────────┬────────┘
                                          │ 6. all chunks done →
                                          ▼    finalize/merge
                                 ┌─────────────────┐
                                 │ Local Upload State   │
                                 │ Store (persists chunk │
                                 │ progress to disk)      │
                                 └─────────────────┘
```

**When you actually draw this in Excalidraw:** draw the main pipeline top-to-bottom like above, but pull the **Chunk Upload Executor** out into its own small zoomed-in diagram showing chunk states (PENDING → UPLOADING → DONE/FAILED) — that's the part most likely to get follow-up questions.

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **Upload Manager** | Entry point — queues upload requests, enforces max concurrent uploads |
| **Compression Stage** | Resizes/compresses the image before it ever touches the network |
| **Chunking Stage** | Splits the file into fixed-size parts (e.g. 1MB chunks) |
| **Chunk Upload Executor** | Uploads each chunk individually, tracks per-chunk success/failure, retries only the failed ones |
| **Local Upload State Store** | Persists which chunks have succeeded, so a killed app can resume instead of restarting |
| **Progress Callback** | Reports overall % back to the app's UI as chunks complete |

### B. Data flow — walking through "Uploading an Image"
```
1. App calls upload(fileUri), gets added to Upload Manager's queue
2. When a concurrency slot is free, Compression Stage resizes/compresses it
3. Chunking Stage splits the compressed file into, say, 5 chunks of 1MB each
4. Chunk Upload Executor uploads chunk 1 → success → chunk 2 → success → ...
   After each chunk succeeds, Local Upload State Store marks it DONE on disk
   and Progress Callback fires (e.g. 40% after chunk 2/5 completes)
5. If chunk 3 fails (network drop), only chunk 3 is retried — 1 and 2
   are NOT re-uploaded
6. Once all chunks succeed, Chunk Upload Executor tells the server to
   finalize/merge them into the complete file
```

### C. Data flow — walking through "Resuming After App Kill"
```
1. App was killed mid-upload; chunks 1-3 of 5 were already DONE
2. App restarts, calls upload() again for the same file (or SDK
   auto-resumes queued uploads on init)
3. Local Upload State Store is checked FIRST — it already knows
   chunks 1-3 succeeded
4. Chunk Upload Executor skips straight to chunk 4 — no wasted
   re-upload of already-successful data
```

### D. Why these choices (tie back to NFRs)
- **Compression before chunking** exists because of **Battery/Data Efficiency** — smaller file means fewer chunks, less data used, faster overall upload.
- **Chunking + per-chunk retry** exists because of **Reliability + Resilience to poor network** — losing connection at 90% shouldn't mean restarting from 0%; only the failed chunk needs retrying.
- **Local Upload State Store (persisted to disk)** exists because of **Reliability** — even a full app kill/crash doesn't lose upload progress, since chunk state lives on disk, not just in memory.
- **Concurrency limit in Upload Manager** exists because of **Resource Efficiency** — uploading 10 photos at once would saturate the connection and make ALL of them slow; capping concurrency (e.g. 2 at a time) keeps things smooth.

### E. One trade-off to mention
> "Chunking adds complexity (tracking per-chunk state, coordinating a server-side merge) versus a single simple upload. For small images this overhead isn't worth it — a smarter design applies chunking only above a size threshold (e.g. >2MB), and does a simple single-shot upload below that."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **Choosing chunk size** | Trade-off: smaller chunks = more resilient to drops but more HTTP overhead; typically 1-5MB is a reasonable middle ground |
| **Uploading in the background (app closed)** | Hand off to WorkManager, same pattern as the Event Logging Library's Upload Manager |
| **Ensuring chunk order/integrity** | Each chunk tagged with an index + checksum; server verifies before merging |
| **Handling duplicate uploads (same file uploaded twice)** | Hash the file content; if a matching hash already exists server-side, skip re-upload entirely |

---

## ✅ Recap in One Line
> Queue → compress → chunk → upload chunk-by-chunk with per-chunk retry, persisting progress to disk so an app kill never means starting over.

---
