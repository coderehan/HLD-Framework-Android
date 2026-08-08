# 📈 HLD — Design Zerodha (Stock Trading App - Android)

**Examples:** Zerodha Kite, Groww, Upstox, Robinhood

> Topic #5 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the user DO?**

| # | Requirement |
|---|---|
| 1 | User can view live stock prices (watchlist, charts) |
| 2 | User can place a buy/sell order (market or limit) |
| 3 | User can view order status (pending, executed, rejected) |
| 4 | User can view portfolio holdings and P&L |
| 5 | User can search and add stocks to a watchlist |
| 6 | User can get notified on order execution / price alerts |

> **Rule of thumb:** two very different worlds live in this app — **live price streaming** (read-heavy, real-time) and **order placement** (write-heavy, must be 100% correct). Call this out early.

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| ⚡ **Low Latency** | Price ticks must reach the screen in milliseconds — stale prices = bad decisions |
| 💰 **Strong Consistency** | An order must execute exactly as intended — no double-buy, no lost trades |
| 🔒 **Reliability** | Order placement must never silently fail or duplicate |
| 🔐 **Security** | Financial data + auth need bank-grade protection (2FA, encrypted storage) |
| 📈 **Scalability** | Market open (9:15 AM) causes a massive spike in both price ticks and orders |

> Skipped **Offline Support** almost entirely — trading fundamentally requires a live connection; at most, cache last-seen prices/portfolio as read-only, clearly marked "stale."

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

> **Same dual-network pattern as Swiggy:** WebSocket for the constant stream of price ticks, Retrofit for one-shot actions like placing an order or fetching portfolio.

### Extended for Zerodha specifics

**Flow 1 — Live Price Streaming**
```
┌──────────────────┐  ticks (100s/sec)  ┌────────────────┐
│ Stock Exchange     │ ─────────────────▶│ Market Data      │
│ Feed (external)     │                   │ Gateway           │
└──────────────────┘                    └────────┬────────┘
                                                   │ publish
                                                   ▼
                                          ┌─────────────────┐
                                          │ Price Broadcast   │
                                          │ (Pub/Sub, per     │
                                          │  symbol channel)  │
                                          └────────┬────────┘
                                                   │ push
                                                   ▼
                                          ┌─────────────────┐
                                          │ Client (subscribed │
                                          │ to watchlist        │
                                          │ symbols only)        │
                                          └─────────────────┘
```

**Flow 2 — Placing an Order**
```
┌─────────┐  1. Place order    ┌─────────────────┐
│ Client  │ ─────────────────▶ │ Order Service     │
└─────────┘                    └────────┬────────┘
                                         │ 2. Validate
                                         ▼    (funds, holdings, limits)
                                ┌─────────────────┐
                                │ Risk & Margin      │
                                │ Check Service       │
                                └────────┬────────┘
                                         │ 3. Forward if valid
                                         ▼
                                ┌─────────────────┐
                                │ Exchange Order     │
                                │ Management System   │
                                │ (OMS)                │
                                └────────┬────────┘
                                         │ 4. Execution confirmed
                                         ▼
                                ┌─────────────────┐
                                │ Portfolio Service   │
                                │ (updates holdings)   │
                                └─────────────────┘
```

**When you actually draw this in Excalidraw:** draw "Price Streaming" and "Order Placement" as two clearly separate diagrams — one is a constant one-way firehose, the other is a careful step-by-step transaction. Mixing them confuses the story.

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **UI Layer** | Renders watchlist, charts, order screen, portfolio |
| **ViewModel** | Holds live price state, order form state, portfolio state |
| **Repository** | Merges live WebSocket prices with cached portfolio/order data |
| **Room (Local DB)** | Caches watchlist symbols, last portfolio snapshot |
| **Market Data Gateway** | Ingests raw price ticks from the stock exchange feed |
| **Price Broadcast (Pub/Sub)** | Fans out ticks only to clients subscribed to that specific symbol |
| **Order Service** | Receives and manages the order lifecycle |
| **Risk & Margin Check Service** | Validates the user has enough funds/holdings before allowing the order |
| **Exchange OMS** | Actually sends the validated order to the stock exchange for execution |
| **Portfolio Service** | Updates holdings and P&L after execution |

### B. Data flow — walking through "Live Price Streaming"
```
1. App opens watchlist screen, subscribes to a list of symbols over WebSocket
2. Market Data Gateway continuously receives ticks from the exchange
3. Price Broadcast fans out each tick only to clients subscribed to that symbol
   (so a user watching 5 stocks doesn't receive updates for 5,000 others)
4. Client receives tick, ViewModel updates price, UI re-renders that row/chart
```

### C. Data flow — walking through "Placing an Order"
```
1. User fills buy/sell form, taps "Place Order"
2. Order Service receives request, first sends it to Risk & Margin Check
3. Risk check verifies: enough funds (buy) or enough holdings (sell)
   → fail: reject immediately, show reason to user
   → pass: forward to Exchange OMS
4. Exchange OMS sends the order to the actual stock exchange
5. On execution confirmation, Portfolio Service updates holdings/P&L
6. User is notified the order was executed
```

### D. Why these choices (tie back to NFRs)
- **Pub/Sub with per-symbol subscription** exists because of **Low Latency + Scalability** — broadcasting every tick to every client would be wasteful; only sending what each user actually watches keeps it fast at scale.
- **Risk & Margin Check before Exchange OMS** exists because of **Strong Consistency + Reliability** — catching an invalid order (insufficient funds) before it ever reaches the exchange avoids a failed/reversed trade later.
- **Separate Order Service from Portfolio Service** exists because of **Reliability** — order placement and portfolio updates are distinct steps; if portfolio update briefly lags, the order itself is still safely recorded.

### E. One trade-off to mention
> "We could let the client apply the buy/sell directly to its local portfolio copy for a snappier feel, but for a trading app, we always wait for the Exchange's confirmed execution before updating anything — showing an unconfirmed trade as 'done' is unacceptable when real money is involved."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **Handling market-open traffic spike** | Auto-scale Market Data Gateway + Order Service, pre-warm connections before 9:15 AM |
| **Ensuring no duplicate orders** | Idempotency key per order request, so a retried tap doesn't place it twice |
| **Reducing chart rendering lag** | Client-side downsampling of tick data for chart display (don't redraw on every single tick) |
| **Order rejected but user unclear why** | Risk & Margin Check returns a specific reason code (insufficient funds, circuit limit, etc.) shown in UI |

---

## ✅ Recap in One Line
> Prices = exchange feed → pub/sub → only-what-you-watch pushed live (speed-first). Orders = validate funds → send to exchange → confirm → update portfolio (correctness-first, no shortcuts).

---
