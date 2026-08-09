# 🏠 HLD — Design Property Listing Service (Android)

**Examples:** Airbnb, 99acres, MagicBricks, Zillow

> Topic #12 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the user DO?**

| # | Requirement |
|---|---|
| 1 | User can search properties by location (map/area) |
| 2 | User can filter by price, type (rent/buy), BHK, amenities |
| 3 | User can view property details (photos, price, owner/agent contact) |
| 4 | Owner/agent can list a new property with photos and details |
| 5 | User can save/shortlist properties |
| 6 | User can get notified when a new matching property is listed (saved search alert) |

> **Rule of thumb:** two very different actors — the **searcher** (read-heavy, needs fast geo-filtered results) and the **lister** (write-heavy, uploads photos + details). Call this out early, same as Swiggy's three-actor split.

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| ⚡ **Low Latency Search** | Map-based/filtered search must return results fast as user pans/zooms |
| 📈 **Scalability** | Millions of listings, heavy concurrent search traffic in popular cities |
| 🗺️ **Geo-Query Efficiency** | "Properties within this map bound" is a different problem than normal DB filtering |
| 🖼️ **Media Handling** | Listings are photo-heavy — fast image loading matters (ties to Image Loading Library topic) |
| 🔔 **Timely Alerts** | Saved-search notifications shouldn't lag hours behind a new listing |

> Skipped strict **Consistency** — a listing being visible a few seconds late to some users is an acceptable trade-off, unlike inventory/payment systems.

---

## 🎨 Step 3: HLD Diagram

### Base Android structure (always start here)
```
                    User
                     |
                     ↓
              UI Layer (Compose)
                     |
                     ↓
                ViewModel
                     |
                     ↓
                Repository
                 /        \
                ↓          ↓
           Room (Local)   Retrofit (Network)
                     |
                     ↓
                  Backend
```

### Extended for Property Listing specifics

**Flow 1 — Searching Properties**
```
┌─────────┐  1. search(bounds,   ┌─────────────────┐
│ Client  │    filters)          │ Search Service     │
│ (map)   │ ──────────────────▶  └────────┬────────┘
└─────────┘                               │ 2. geo + filter query
                                           ▼
                                  ┌─────────────────┐
                                  │ Geo-Indexed Search  │
                                  │ Engine (Elasticsearch│
                                  │ with geo-queries)      │
                                  └────────┬────────┘
                                           │ 3. matching listing IDs
                                           ▼
                                  ┌─────────────────┐
                                  │ Listing Cache        │
                                  │ (Redis, listing        │
                                  │  summary data)          │
                                  └─────────────────┘
```

**Flow 2 — Creating a Listing**
```
┌─────────┐  1. Create listing  ┌─────────────────┐
│ Owner/  │ ───────────────────▶│ Listing Service     │
│ Agent   │                      └────────┬────────┘
└─────────┘                               │ 2. save details
                                           ▼
                                  ┌─────────────────┐
                                  │ Listing DB           │
                                  └────────┬────────┘
                                           │ 3. upload photos
                                           ▼
                                  ┌─────────────────┐
                                  │ Media/CDN Storage    │
                                  └────────┬────────┘
                                           │ 4. index for search
                                           ▼
                                  ┌─────────────────┐
                                  │ Search Indexer        │
                                  │ (pushes into geo-index) │
                                  └────────┬────────┘
                                           │ 5. check saved searches
                                           ▼
                                  ┌─────────────────┐
                                  │ Alert Matching        │
                                  │ Service                 │
                                  └─────────────────┘
```

**When you actually draw this in Excalidraw:** draw "Search" and "Create Listing" as two separate diagrams, same pattern as e-commerce. The Search Service's geo-index is the centerpiece worth zooming into.

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **UI Layer** | Renders map, filters, listing cards, detail screen |
| **ViewModel** | Holds search results, filter state, map bounds |
| **Repository** | Caches recent search results locally, decides when to re-query |
| **Room (Local DB)** | Caches last search results + shortlisted properties for offline viewing |
| **Search Service** | Accepts search/filter requests, talks to the geo-index |
| **Geo-Indexed Search Engine** | Efficiently answers "what listings fall within this map area + filters" |
| **Listing Service** | Handles creating/editing listings |
| **Media/CDN Storage** | Stores and serves listing photos fast |
| **Search Indexer** | Keeps the geo-index in sync whenever a listing is created/updated |
| **Alert Matching Service** | Checks new listings against saved searches, triggers notifications |

### B. Data flow — walking through "Searching Properties"
```
1. User pans the map or applies filters (price, BHK)
2. Client sends the current map bounds + filters to Search Service
3. Search Service queries the Geo-Indexed Search Engine — this is
   NOT a normal SQL "WHERE lat BETWEEN..." query; a proper geo-index
   (like Elasticsearch geo_bounding_box or a geohash-based index)
   is used because naive lat/lng range queries don't scale well
4. Matching listing IDs come back, summary data is fetched from
   Listing Cache (Redis) for fast rendering
5. Results shown as pins on map / cards in list
```

### C. Data flow — walking through "Creating a Listing"
```
1. Owner fills in property details + uploads photos
2. Listing Service saves the core details to Listing DB
3. Photos are uploaded to Media/CDN Storage (same pattern as the
   Image Upload Library topic — could literally reuse that library)
4. Search Indexer picks up the new listing and adds it to the
   geo-index so it becomes searchable
5. Alert Matching Service checks: does this listing match anyone's
   saved search? → if yes, trigger a notification to them
```

### D. Why these choices (tie back to NFRs)
- **Geo-Indexed Search Engine (not plain SQL)** exists because of **Low Latency Search + Geo-Query Efficiency** — "find everything within this visible map rectangle, filtered by price/BHK" needs spatial indexing (geohash/quadtree-based) to stay fast at scale; a plain relational WHERE clause degrades badly.
- **Listing Cache (Redis)** exists because of **Scalability** — popular searches (e.g. "2BHK in Bangalore") get hit repeatedly; caching summary data avoids re-fetching full records from the DB every time.
- **Async Search Indexer (not synchronous)** exists because listing creation shouldn't be blocked waiting for the search index to update — the owner gets a fast "listing created" response, and indexing happens right after.

### E. One trade-off to mention
> "Indexing asynchronously means a brand-new listing might take a few seconds to appear in search results — acceptable here since it's not time-critical, unlike say a stock price. If instant searchability were a hard requirement, we'd need synchronous indexing at the cost of slower listing creation."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **How does geo-search actually scale?** | Geohashing/quadtree indexing divides the map into cells; querying a visible area only touches relevant cells, not the whole dataset |
| **Handling stale listings (already sold/rented)** | TTL-based re-verification prompts to owners, or auto-expire after N days of inactivity |
| **Fast image loading for listing photos** | Reuse the Image Loading Library pattern — memory/disk cache + downsampling |
| **Duplicate listings (same property posted twice)** | Fuzzy-match on address + photos at listing-creation time, flag for review |

---

## ✅ Recap in One Line
> Search = geo-indexed query (not plain SQL) → cached summaries for speed. Create = save details → upload media → async-index for search → check saved-search alerts.

---
