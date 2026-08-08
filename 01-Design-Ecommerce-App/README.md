# 🛒 HLD — Design E-Commerce App (Android)

> Topic #1 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the user DO?**

| # | Requirement |
|---|---|
| 1 | User can browse products (home feed, categories) |
| 2 | User can search products with filters (price, brand, rating) |
| 3 | User can view product details (images, price, reviews) |
| 4 | User can add products to cart / wishlist |
| 5 | User can place an order and pay |
| 6 | User can track order status and get notifications |

> **Rule of thumb:** if the interviewer didn't limit scope, pick the 4-6 you just read above — that's a safe, complete slice of an e-commerce app.

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| 🚀 **Performance** | Product feed and images must load fast — this is what users see first |
| 📈 **Scalability** | Sale events (Big Billion Days style) spike traffic 10-100x |
| 💰 **Consistency** | Cart + inventory + payment must never go wrong — no overselling |
| 📴 **Offline Support** | Browse cached products even on flaky network |
| 🔋 **Battery Efficiency** | No unnecessary background syncing draining battery |

> Only picked 5 — not every NFR applies to every app. Skip ones that don't matter (e.g. real-time location isn't relevant here).

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

### Extended for E-Commerce specifics
```
   ┌─────────┐     1. Fetch feed      ┌──────────────┐
   │  Client │ ────────────────────▶  │  API Gateway │
   │ (App)   │                        └──────┬───────┘
   └────┬────┘                               │
        │ 6. Show cached                     │ 2. Route
        │    data instantly                  ▼
        ▼                          ┌──────────────────┐
   ┌──────────┐                    │  Product Service  │
   │  Room DB │ ◀── 5. Cache ────  └─────────┬─────────┘
   │ (cache)  │       response               │ 3. Query
   └──────────┘                              ▼
                                   ┌──────────────────┐
                                   │  Product DB +     │
                                   │  Cache (Redis)    │
                                   └──────────────────┘

   Separately:
   ┌──────────┐   Add to Cart   ┌────────────────┐
   │  Client  │ ──────────────▶ │  Cart Service   │
   └──────────┘                 └────────┬────────┘
                                          │ Checkout
                                          ▼
                                 ┌─────────────────┐
                                 │ Order Service    │
                                 └────────┬────────┘
                                          │ Charge
                                          ▼
                                 ┌─────────────────┐
                                 │ Payment Service  │
                                 └────────┬────────┘
                                          │ Confirm
                                          ▼
                                 ┌─────────────────┐
                                 │ Notification     │
                                 │ Service (Push)   │
                                 └─────────────────┘
```

**When you actually draw this in Excalidraw:** keep it to ONE flow at a time — draw "Browse Products" flow first, explain it, then draw "Checkout" flow as a second, separate mini-diagram. Don't try to cram both into one messy diagram.

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **UI Layer** | Shows product feed, cart, checkout screens |
| **ViewModel** | Holds UI state (loading/success/error), survives rotation |
| **Repository** | Decides: return cached data or fetch fresh from network |
| **Room (Local DB)** | Stores last-fetched products for instant load + offline browsing |
| **Product Service** | Backend service that serves product listings/details |
| **Cart Service** | Manages what's in the user's cart |
| **Order Service** | Creates the order, talks to Payment Service |
| **Payment Service** | Handles charging the user securely |
| **Redis Cache** | Speeds up repeated product reads without hitting DB every time |
| **Notification Service** | Sends push updates ("Order shipped!") |

### B. Data flow — walking through "Browse Products"
```
1. User opens app
2. UI asks ViewModel for product list
3. ViewModel asks Repository
4. Repository checks Room cache first
   → if fresh cache exists: return it instantly (feels fast)
   → if stale/empty: call Retrofit → API Gateway → Product Service
5. Product Service checks Redis cache
   → hit: return fast
   → miss: query Product DB, then populate Redis
6. Response flows back up, Repository saves it in Room, UI updates
```

### C. Data flow — walking through "Checkout"
```
1. User taps "Place Order" in Cart
2. Cart Service sends items to Order Service
3. Order Service calls Payment Service to charge the user
4. On success: Order Service marks order as CONFIRMED
5. Notification Service pushes "Order Placed" to the user
```

### D. Why these choices (tie back to NFRs)
- **Room cache** exists because of the **Performance + Offline** NFRs — user sees something instantly, even on bad network.
- **Redis cache** exists because of **Scalability** — during a sale, thousands of users view the same trending products; no need to hit the DB every time.
- **Separate Order + Payment services** exist because of **Consistency** — payment failures shouldn't corrupt order state, so they're handled as distinct, coordinated steps.

### E. One trade-off to mention
> "We could store cart data only on the server, but keeping a local cart copy (Room) means the user can add/remove items offline and it syncs once back online — better UX at the cost of needing conflict resolution later."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **Handling sale-day traffic spikes** | Redis cache + horizontal scaling of Product Service + CDN for images |
| **Preventing overselling** | Inventory check + lock at checkout time, not at "add to cart" time |
| **Faster image loading** | CDN + image compression + placeholder/lazy loading in UI |
| **Offline cart sync conflicts** | Last-write-wins, or merge quantities with a sync timestamp |

---

## ✅ Recap in One Line
> Browse = Room cache → Redis cache → DB (speed). Checkout = Cart → Order → Payment → Notify (consistency).

---

**Status:** ✔️ Completed — Topic #1 of the HLD series.
