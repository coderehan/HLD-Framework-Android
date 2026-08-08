# 🍔 HLD — Design Swiggy (Food Delivery App - Android)

**Examples:** Zomato, Uber Eats, DoorDash

> Topic #4 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the user DO?**

| # | Requirement |
|---|---|
| 1 | User can browse restaurants and menus (based on location) |
| 2 | User can add items to cart and place an order |
| 3 | User can make a payment for the order |
| 4 | System can match a delivery partner to the order |
| 5 | User can track delivery partner's live location on a map |
| 6 | User can get notified at each order stage (accepted, picked up, delivered) |

> **Rule of thumb:** three actors here, not one — Customer, Restaurant, Delivery Partner. Say this out loud early; it shapes the whole design.

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| ⚡ **Low Latency** | Live location must update on the map smoothly, every few seconds |
| 📈 **Scalability** | Lunch/dinner peak hours = massive spike in orders and location pings |
| 💰 **Consistency** | Order + payment + inventory (restaurant availability) must never conflict |
| 🔋 **Battery Efficiency** | Delivery partner's app sends location constantly — must not drain battery |
| 🔒 **Reliability** | An order should never silently get lost between placement and delivery |

> Skipped strict **Offline Support** here — unlike a chat app, placing an order fundamentally needs network; light caching (menus) is enough.

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
           Room (Local)   Retrofit + WebSocket (Network)
                     |
                     ↓
                  Backend
```

> **Key difference here:** we need BOTH — Retrofit for normal actions (browse, order, pay) AND a persistent connection (WebSocket) for live location updates, same reasoning as the chat app.

### Extended for Swiggy specifics

**Flow 1 — Placing an Order**
```
┌─────────┐  1. Place order   ┌─────────────────┐
│ Customer│ ────────────────▶ │ Order Service     │
│ Client  │                   └────────┬────────┘
└─────────┘                            │ 2. Reserve items
                                        ▼
                               ┌─────────────────┐
                               │ Restaurant Service │
                               │ (checks menu/stock) │
                               └────────┬────────┘
                                        │ 3. Charge
                                        ▼
                               ┌─────────────────┐
                               │ Payment Service   │
                               └────────┬────────┘
                                        │ 4. Find rider
                                        ▼
                               ┌─────────────────┐
                               │ Delivery Matching │
                               │ Service            │
                               └─────────────────┘
```

**Flow 2 — Live Location Tracking**
```
┌──────────────┐  every few sec   ┌────────────────┐
│ Delivery      │ ───────────────▶│ Location Gateway │
│ Partner Client│  (lat, lng)      │ (WebSocket)      │
└──────────────┘                  └────────┬────────┘
                                            │ store latest position
                                            ▼
                                   ┌─────────────────┐
                                   │ Location Cache    │
                                   │ (Redis, keyed by  │
                                   │  order/rider ID)  │
                                   └────────┬────────┘
                                            │ push to
                                            ▼
                                   ┌─────────────────┐
                                   │ Customer Client   │
                                   │ (map updates live) │
                                   └─────────────────┘
```

**When you actually draw this in Excalidraw:** draw "Place Order" and "Live Tracking" as two separate small diagrams — one is request/response heavy, the other is streaming/real-time. Mixing them in one diagram gets messy fast.

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **UI Layer** | Renders restaurant list, menu, cart, live tracking map |
| **ViewModel** | Holds order state, cart state, tracking state |
| **Repository** | Combines cached menu data (Room) with live network/order data |
| **Room (Local DB)** | Caches restaurant/menu data for quick reload |
| **Order Service** | Creates and manages the lifecycle of an order |
| **Restaurant Service** | Owns menu data, checks item availability |
| **Payment Service** | Handles secure charging |
| **Delivery Matching Service** | Finds the nearest available delivery partner |
| **Location Gateway (WebSocket)** | Receives constant location pings from delivery partner, pushes to customer |
| **Location Cache (Redis)** | Stores each active order's latest rider position for fast reads |

### B. Data flow — walking through "Place Order"
```
1. Customer adds items to cart and taps "Place Order"
2. Order Service creates the order, calls Restaurant Service to confirm availability
3. Payment Service charges the customer
4. Order Service marks order CONFIRMED, calls Delivery Matching Service
5. Delivery Matching Service assigns the nearest available rider
6. Customer + Restaurant + Rider all get notified of the confirmed order
```

### C. Data flow — walking through "Live Tracking"
```
1. Delivery partner's app sends (lat, lng) every few seconds over WebSocket
2. Location Gateway writes the latest position into Redis (keyed by order ID)
3. Customer's app, subscribed to the same order's channel, receives the update
4. Customer's map marker moves smoothly to the new position
```

### D. Why these choices (tie back to NFRs)
- **WebSocket for location** exists because of **Low Latency** — polling every few seconds via Retrofit would be slower and less battery-friendly than one open push connection.
- **Redis for latest location** exists because of **Scalability** — we only ever need the *latest* position per order, not history, so a fast key-value cache beats writing every ping to a DB.
- **Reserve items before charging** (Restaurant Service check happens before Payment) exists because of **Consistency** — avoids charging a customer for an item that just went out of stock.

### E. One trade-off to mention
> "Sending location every 2-3 seconds is smooth but drains the delivery partner's battery faster. A common trade-off is adaptive ping frequency — send more often when moving fast/near delivery, less often when stationary — balancing Low Latency against Battery Efficiency."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **Peak hour order surge** | Auto-scale Order/Restaurant services, queue orders if a restaurant is overwhelmed |
| **Rider matching algorithm** | Nearest available rider by geohash/grid search, factoring in current load |
| **Reducing battery drain on rider app** | Adaptive location ping interval + batching updates |
| **Handling payment failure mid-order** | Rollback: release reserved items, mark order FAILED, notify customer to retry |

---

## ✅ Recap in One Line
> Order = reserve → charge → match rider (consistency-first, step by step). Tracking = rider pings via WebSocket → Redis latest-position cache → pushed live to customer (speed-first).

---
