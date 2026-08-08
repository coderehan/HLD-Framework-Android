# 🖼️ HLD — Design Image Loading Library (Android)

**Examples:** Glide, Coil, Picasso, Fresco

> Topic #10 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.
>
> ⚠️ **Note:** The mirror image of the Image Upload Library — that one was "send a big file out reliably." This one is "fetch, decode, cache, and display a large image FAST, without leaking memory or crashing a RecyclerView." The big new ideas here: **multi-level caching**, **request deduplication**, and **lifecycle-awareness**.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the (developer/host app) DO with this library?**

| # | Requirement |
|---|---|
| 1 | Developer can load an image into an ImageView with one line: `load(url).into(imageView)` |
| 2 | Library can show a placeholder while loading and an error image on failure |
| 3 | Library can cache images in memory AND on disk to avoid re-downloading |
| 4 | Library can resize/downsample images to match the target view's size |
| 5 | Library can cancel a load automatically when the view is recycled/destroyed |
| 6 | Library can transform images (circle crop, blur, rounded corners) |

> **Rule of thumb:** the "user" is a developer calling `ImageLoader.load(url).into(view)` inside a RecyclerView adapter binding hundreds of times per second while scrolling — that scrolling context is what makes this topic hard.

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| ⚡ **Performance** | Must render smoothly inside a fast-scrolling RecyclerView — no jank |
| 🧠 **Memory Efficiency** | Full-resolution images can easily OOM-crash the app if not downsampled/cached carefully |
| 🔁 **No Redundant Work** | Scrolling up/down often re-requests the same URL — must not re-download or re-decode needlessly |
| 🔋 **Battery/Data Efficiency** | Disk cache avoids re-downloading the same image across app sessions |
| 🧵 **Lifecycle Safety** | A view recycled mid-load must not receive a stale image or leak memory |

> Skipped **Reliability/Resumability** here (unlike the Upload Library) — if a single image load fails, we just show an error placeholder and let the user scroll/retry; there's no partial-state to protect.

---

## 🎨 Step 3: HLD Diagram

### Base structure (SDK-style — pipeline with a cache lookup up front)
```
      load(url).into(imageView)
                    |
                    ↓
             Request Manager
        (attaches to view's lifecycle)
                    |
                    ↓
             Memory Cache (LRU)
              hit?  /      \  miss
                   ↓          ↓
             Return image   Disk Cache
             instantly       hit? / \ miss
                                 ↓     ↓
                          Return &   Network
                          promote to  Fetch
                          memory cache    |
                                          ↓
                                   Decode + Downsample
                                          |
                                          ↓
                              Store in Disk + Memory Cache
                                          |
                                          ↓
                                Deliver to ImageView
```

### Extended internal flow
```
┌──────────┐ 1. load(url)       ┌─────────────────┐
│ App/     │ ──────────────────▶│ Request Manager    │
│ Adapter  │  .into(imageView)   │ (tracks view       │
└──────────┘                    │  lifecycle + tag)   │
                                 └────────┬────────┘
                                          │ 2. check for duplicate
                                          ▼    in-flight request
                                 ┌─────────────────┐
                                 │ Request               │
                                 │ Deduplication          │
                                 │ (same URL already       │
                                 │  loading? attach to it)   │
                                 └────────┬────────┘
                                          │ 3. check cache chain
                                          ▼
                            ┌──────────────────────────┐
                            │ Memory Cache (LRU)          │
                            └───────┬──────────────────┘
                              hit   │   miss
                          ┌─────────┘   └─────────┐
                          ▼                        ▼
                  Return instantly        ┌─────────────────┐
                  (no decode needed)       │ Disk Cache         │
                                           └───────┬────────┘
                                             hit    │   miss
                                        ┌───────────┘   └───────────┐
                                        ▼                            ▼
                              Decode from disk            ┌─────────────────┐
                              (fast, no network)            │ Network Fetch      │
                                                            └────────┬────────┘
                                                                     │
                                                                     ▼
                                                            ┌─────────────────┐
                                                            │ Decode +            │
                                                            │ Downsample           │
                                                            │ (to target view size)│
                                                            └────────┬────────┘
                                                                     │
                                                                     ▼
                                                     Store in Disk Cache + Memory Cache
                                                                     │
                                                                     ▼
                                                     4. Deliver to ImageView
                                                        (only if view is still
                                                         alive & wants this URL)
```

**When you actually draw this in Excalidraw:** draw the **3-level cache lookup** (Memory → Disk → Network) as the centerpiece — that vertical fallback chain is the single most important thing to get across.

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **Request Manager** | Entry point; binds the load request to the target view's lifecycle |
| **Request Deduplication** | If the same URL is already being fetched (common while scrolling), attach the new request to the existing one instead of firing a second download |
| **Memory Cache (LRU)** | Fastest layer — holds recently-used bitmaps in RAM, evicts least-recently-used when full |
| **Disk Cache** | Slower than memory but survives app restarts — avoids re-downloading across sessions |
| **Decode + Downsample** | Decodes the raw bytes into a bitmap sized to fit the target ImageView (never load a 4000x3000 photo into a 100x100 thumbnail) |
| **Lifecycle binding** | Cancels the load automatically if the view is destroyed/recycled before it completes |

### B. Data flow — walking through "Loading an Image"
```
1. Adapter calls load(url).into(imageView) while binding a RecyclerView row
2. Request Manager tags this request with the view's lifecycle
3. Request Deduplication checks: is this exact URL already being fetched
   for another view? → if yes, just attach a callback to that existing request
4. Check Memory Cache
   → hit: return the bitmap instantly, done (no decode, no I/O)
   → miss: check Disk Cache
5. Check Disk Cache
   → hit: decode from local file (fast, no network), then promote to Memory Cache
   → miss: go to Network Fetch
6. Network Fetch downloads the raw image bytes
7. Decode + Downsample resizes it to match the target ImageView's dimensions
   (critical — decoding full resolution into a tiny thumbnail wastes memory)
8. Result is stored in both Disk Cache and Memory Cache for next time
9. Before actually setting the bitmap, Request Manager checks: is this
   view still on screen and still wanting THIS url? → if the view was
   recycled and now wants a different image, DROP this result
```

### C. Why these choices (tie back to NFRs)
- **3-level cache (Memory → Disk → Network)** exists because of **Performance + Battery/Data Efficiency** — always try the fastest source first; only fall back to network when truly necessary.
- **Downsampling to target view size** exists because of **Memory Efficiency** — this is the #1 cause of OOM crashes in naive image-loading code; never keep a full-res bitmap in memory for a tiny thumbnail.
- **Request Deduplication** exists because of **No Redundant Work** — fast-scrolling a RecyclerView can trigger the same URL request many times in a second; deduping avoids wasted network/decode work.
- **Lifecycle-aware cancellation** exists because of **Lifecycle Safety** — without this, a slow network response for a long-scrolled-past view could deliver a wrong/stale image or leak the view's Context.

### D. One trade-off to mention
> "A larger memory cache means fewer re-decodes (faster scrolling) but eats into the app's available RAM, increasing OOM risk elsewhere. Libraries like Glide size the memory cache dynamically as a fraction of available app memory rather than a fixed constant, to balance this per-device."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **Avoiding OOM crashes** | Downsample based on target view size + use `inBitmap` reuse pools to avoid extra allocations |
| **Handling the same view rapidly requesting different URLs (fast scroll)** | Tag each view with the "latest wanted URL"; on delivery, discard results for stale tags |
| **Cache eviction strategy** | LRU (Least Recently Used) for memory; disk cache typically LRU by size cap too |
| **Supporting GIFs/animated images** | Separate decoder pipeline that produces a Drawable capable of animating frames, instead of a static Bitmap |

---

## ✅ Recap in One Line
> Dedupe in-flight requests → check Memory Cache → Disk Cache → Network, in that order → downsample to view size → cache the result both places → deliver only if the view still wants it.

---
