# ✈️ HLD — Design Flight Aggregation System (Android)

**Examples:** Skyscanner, Google Flights, Kayak, ixigo

> Topic #13 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.
>
> ⚠️ **Note:** This is NOT a booking system — it's a **search/comparison** engine. The core challenge: call MANY airline/OTA provider APIs at once, each with different speed/reliability, and show a merged, ranked result FAST — without waiting for the slowest one.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the user DO?**

| # | Requirement |
|---|---|
| 1 | User can search flights by origin, destination, date |
| 2 | User can see results from multiple airlines/providers in one list |
| 3 | User can sort/filter by price, duration, stops, airline |
| 4 | User can view fare details before being redirected to book |
| 5 | System can show results progressively as providers respond (not all-or-nothing) |
| 6 | System can cache recent searches to serve repeat queries fast |

> **Rule of thumb:** the interviewer wants to see you handle **fan-out to multiple external APIs with wildly different response times** — that's the whole point of this topic.

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| ⚡ **Low Latency (perceived)** | Users expect results within 1-2 sec, even though real providers take longer |
| 🔀 **Fault Tolerance** | One slow/dead provider shouldn't block or crash the whole search |
| 📈 **Scalability** | Popular routes get searched very frequently — caching matters a lot |
| 🔁 **Freshness vs Speed Trade-off** | Prices change fast, but showing SOMETHING quickly beats waiting for perfectly fresh data |
| 🔧 **Extensibility** | Adding a new airline/provider integration shouldn't require touching the core search flow |

> Skipped **Strict Consistency** entirely — this is a search/comparison layer; the actual booking (with real consistency needs) happens on the airline/OTA's own system, not here.

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
           Room (Local)   Retrofit + WebSocket/SSE (Network)
                     |
                     ↓
                  Backend
```

> **Key difference here:** WebSocket/Server-Sent-Events is used so results can stream in progressively as each provider responds — same underlying idea as the chat app's live push, applied to search results instead of messages.

### Extended for Flight Aggregation specifics
```
┌─────────┐  1. search(origin,   ┌─────────────────┐
│ Client  │    dest, date)        │ Search              │
│         │ ─────────────────────▶│ Orchestrator          │
└─────────┘                       └────────┬────────┘
     ▲                                     │ 2. check cache first
     │ 6. results stream in                ▼
     │    progressively             ┌─────────────────┐
     │    over SSE/WebSocket         │ Result Cache        │
     │                               │ (Redis, keyed by      │
     │                               │  route+date)            │
     │                               └────────┬────────┘
     │                             cache miss  │
     │                                         ▼
     │                               ┌─────────────────┐
     │                               │ Fan-out to Providers  │
     │                               │ (parallel calls)        │
     │                               └───┬────┬────┬───────┘
     │                                   ▼    ▼    ▼
     │                             ┌─────┐┌─────┐┌─────┐
     │                             │Air- ││Air- ││ OTA  │
     │                             │line ││line ││ API   │
     │                             │ A   ││ B   ││       │
     │                             └──┬──┘└──┬──┘└──┬──┘
     │                                │      │      │
     │                                ▼      ▼      ▼
     │                        3. each provider responds
     │                           independently, at its own speed
     │                                │      │      │
     │                                ▼      ▼      ▼
     └────────────────────  4. Result Merger & Ranker
                              (normalizes formats, sorts,
                               streams each batch to client
                               as it arrives)
                                        │
                                        ▼
                              5. cache the final merged
                                 result for next search
```

**When you actually draw this in Excalidraw:** the **fan-out to multiple providers happening in parallel, with results streaming back independently** is the entire story here — make that the centerpiece, not a side detail.

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **UI Layer** | Renders the flight list, updates it as new results stream in |
| **ViewModel** | Holds the growing result list, sort/filter state |
| **Repository** | Subscribes to the streaming search response, updates local state incrementally |
| **Search Orchestrator** | Entry point — checks cache, else triggers fan-out to providers |
| **Result Cache (Redis)** | Stores recent search results keyed by route+date, short TTL (prices go stale fast) |
| **Provider Adapters (Airline A/B, OTA)** | Each wraps a specific external API, normalizing its response into a common format |
| **Result Merger & Ranker** | Combines results from all providers, sorts by price/duration, streams updates to client |

### B. Data flow — walking through "Searching Flights"
```
1. User searches DEL → BLR on a given date
2. Search Orchestrator first checks Result Cache
   → cache hit (recent identical search): return instantly, done
   → cache miss: continue
3. Orchestrator fans out parallel calls to every configured provider
   (Airline A, Airline B, OTA API, etc.) — all at once, not sequentially
4. As EACH provider responds (they finish at different times —
   Airline A might reply in 300ms, OTA API might take 3 seconds):
   - Result Merger & Ranker normalizes that provider's response
     into a common Flight object shape
   - Merged/re-ranked partial results are pushed to the client
     immediately over the streaming connection
5. Client's UI updates progressively — user sees SOME flights within
   a second, and the list keeps growing/re-sorting as more arrive
6. Once all providers respond (or a timeout is hit, whichever first),
   the final merged result is cached for the next identical search
```

### C. Why these choices (tie back to NFRs)
- **Parallel fan-out (not sequential calls)** exists because of **Low Latency (perceived)** — calling providers one after another would mean total wait time = sum of all provider latencies; parallel calls mean wait time = slowest single provider.
- **Streaming partial results (not wait-for-all)** exists because of **Low Latency + Fault Tolerance** — if one provider is slow or down, the user still sees results from the fast ones immediately, instead of staring at a blank screen.
- **Short-TTL cache** exists because of **Scalability**, balanced against **Freshness** — popular routes get cached briefly to reduce provider load, but the TTL is short (e.g. 2-5 min) since flight prices change often.
- **Provider Adapter pattern (each provider isolated)** exists because of **Extensibility** — adding a new airline integration means writing one new adapter that outputs the common Flight format, without touching the orchestrator or ranker at all.

### D. One trade-off to mention
> "Streaming partial results gives a faster *perceived* experience, but means the initial sort order can shift as slower providers' results come in (a cheaper flight might appear after the list is already showing). A common mitigation is a short 'settling' delay (e.g. 500ms) or clearly marking the list as 'still loading more options.'"

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **Handling a provider that's consistently down** | Circuit breaker pattern — stop calling it temporarily after repeated failures, retry after a cooldown |
| **Normalizing very different provider response formats** | Each Provider Adapter maps its raw response to one canonical internal Flight model |
| **Preventing duplicate flights shown from 2 providers** | Dedupe by flight number + departure time when merging results |
| **Reducing redundant provider calls for popular routes** | Cache + optionally pre-fetch popular routes proactively (e.g. top 100 routes refreshed every few minutes in the background) |

---

## ✅ Recap in One Line
> Check cache first → fan out to all providers in parallel → stream and merge results as each responds (fast providers shown first) → cache the final merged result briefly for the next identical search.

---
