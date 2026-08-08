# 🛍️ HLD — Design Cart & Wishlist (Android)

**Examples:** Amazon Cart/Wishlist, Flipkart Cart, Myntra Wishlist

> Topic #11 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.
>
> ⚠️ **Note:** This is a *feature-level* HLD, not a full app — think of it as a zoomed-in version of the "Cart" piece from the E-Commerce topic. The interesting problem here isn't the UI, it's: **the same cart must work seamlessly whether the user is offline, logs in on a new device, or has stale data** — that's the crux of the design.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the user DO?**

| # | Requirement |
|---|---|
| 1 | User can add/remove an item from cart or wishlist |
| 2 | User can update quantity of a cart item |
| 3 | User can move an item between cart and wishlist |
| 4 | Cart/wishlist should work offline and sync once back online |
| 5 | Cart/wishlist should be available across devices (login on phone A and B, same cart) |
| 6 | System should reflect real-time price/stock changes when viewing cart |

> **Rule of thumb:** cart and wishlist are basically the same data structure (a list of product references tied to a user) with different behavior on top — say this out loud, it shows you're not over-engineering two separate systems.

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| 📴 **Offline-First** | Adding to cart must feel instant even with no network |
| 🔁 **Multi-Device Consistency** | Cart added on phone should show up on tablet/web without user action |
| ⚡ **Low Latency** | Add/remove must reflect in UI instantly — no spinner for a tap |
| 💰 **Eventual Consistency (not strict)** | A short sync delay across devices is acceptable; it does NOT need bank-grade consistency like payments |
| 🔧 **Conflict Resolution** | Handle the case where the same item was changed on two devices while offline |

> Compare to the Payment/Order NFRs from the E-Commerce topic — those needed **strict** consistency. Cart is different: it's OK if two devices briefly disagree, as long as they converge.

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

### Extended for Cart & Wishlist specifics

**Flow 1 — Adding an Item (Offline-First)**
```
┌─────────┐  1. Add to cart    ┌─────────────────┐
│ Client  │ ─────────────────▶ │ Local Cart Store   │
│ (tap)   │   (instant, no      │ (Room, source of    │
└─────────┘    network wait)    │  truth for UI)        │
                                └────────┬────────┘
                                         │ 2. UI updates
                                         │    IMMEDIATELY
                                         │ 3. queue a sync op
                                         ▼    in background
                                ┌─────────────────┐
                                │ Sync Queue          │
                                │ (WorkManager)         │
                                └────────┬────────┘
                                         │ 4. when online
                                         ▼
                                ┌─────────────────┐
                                │ Cart Service          │
                                │ (backend, source of    │
                                │  truth across devices)   │
                                └─────────────────┘
```

**Flow 2 — Multi-Device Sync**
```
Device A adds item          Device B (different session)
       │                              │
       ▼                              │
Local Cart Store (Room)                │
       │ sync                         │
       ▼                              │
Cart Service (backend) ────────────────┘
       │                         2. Device B polls/subscribes
       │ 1. persists as              for cart changes
       │    source of truth          │
       ▼                              ▼
Cart Service pushes update ──▶ Device B's Local Cart Store
(via push notification or               updates, UI refreshes
 next app-open sync)
```

**When you actually draw this in Excalidraw:** draw "Offline Add" and "Multi-Device Sync" as two separate diagrams, same pattern as before — one shows instant local write + background sync, the other shows how that syncs back out to a second device.

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **UI Layer** | Renders cart/wishlist list, quantity controls, "move to wishlist" action |
| **ViewModel** | Holds cart state, observes Room for live updates |
| **Repository** | Single source of truth logic — always reads from Room, writes go local-first then sync |
| **Local Cart Store (Room)** | The actual source of truth for the UI — every add/remove writes here FIRST |
| **Sync Queue (WorkManager)** | Holds pending changes to push to the backend once online |
| **Cart Service (backend)** | The cross-device source of truth; merges changes from all devices |

### B. Data flow — walking through "Add to Cart (Offline)"
```
1. User taps "Add to Cart"
2. Item is written to Local Cart Store (Room) immediately
3. ViewModel observes Room via a reactive query (Flow) — UI updates
   instantly, no waiting on network
4. A sync operation is queued in WorkManager
5. WorkManager waits for network availability, then pushes the
   change to Cart Service
6. Cart Service persists it as the new source of truth
```

### C. Data flow — walking through "Sync Across Devices"
```
1. Device A's change reaches Cart Service (as above)
2. Device B, on next app open (or via a lightweight push signal),
   asks Cart Service "what's changed since my last sync timestamp?"
3. Cart Service returns the delta (new/changed/removed items)
4. Device B's Repository merges this delta into its Local Cart Store
5. Device B's UI updates to reflect Device A's change
```

### D. Why these choices (tie back to NFRs)
- **Room as the UI's source of truth (not the network)** exists because of **Offline-First + Low Latency** — the UI never waits on a network call to reflect a tap; it reads local data that's updated instantly.
- **WorkManager for sync** exists because of **Offline-First + Battery Efficiency** — syncing happens respecting network/battery constraints, and retries automatically without the app needing to be open.
- **Delta sync (only changes since last timestamp)** exists because of **Low Latency + Efficiency** — no need to re-download the entire cart every time; just what changed.
- **Eventual consistency across devices** (not instant/strict) exists because cart data doesn't need bank-grade correctness — unlike a Payment/Order, a few seconds of lag between devices is a fine trade-off for a much simpler, offline-friendly design.

### E. One trade-off to mention
> "Writing locally first and syncing later means two devices can briefly disagree — e.g. Device A removes an item while Device B, offline, still shows it. A simple resolution: last-write-wins by timestamp. For quantity changes specifically, you could instead merge (sum/max) rather than overwrite, depending on product decision."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **Conflict when same item edited on 2 offline devices** | Last-write-wins by timestamp, or merge strategy for quantities |
| **Guest user cart merging into account on login** | On login, merge local guest cart into the account's server cart (usually union, ask user if quantities conflict) |
| **Showing live price/stock changes in cart** | Cart Service revalidates price/stock against Product Service when cart is fetched/viewed, flags "price changed" items |
| **Reducing sync payload size at scale** | Delta sync using a `lastSyncedAt` timestamp instead of full cart re-fetch |

---

## ✅ Recap in One Line
> Every add/remove writes to Room FIRST (instant UI, offline-friendly) → queued and synced to backend via WorkManager → backend is the cross-device source of truth → other devices pull deltas and merge, with last-write-wins for simple conflict resolution.

---
