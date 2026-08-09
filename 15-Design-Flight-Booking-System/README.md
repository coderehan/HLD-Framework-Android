# 🎫 HLD — Design Flight Booking System (Android)

**Examples:** MakeMyTrip, IndiGo App, Cleartrip

> Topic #15 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.
>
> ⚠️ **Note:** Compare this to the Flight Aggregation System topic — that one was "search and compare" (read-heavy, no strict consistency needed). This one is "reserve one seat out of a limited inventory and get paid for it correctly" — a completely different, write-heavy, consistency-critical problem. Say this contrast out loud if both come up.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the user DO?**

| # | Requirement |
|---|---|
| 1 | User can select a flight and choose seat(s) |
| 2 | User can hold a seat temporarily while completing payment |
| 3 | User can pay and confirm the booking |
| 4 | User can view/cancel an existing booking |
| 5 | System can prevent the same seat from being double-booked |
| 6 | User can get a confirmation (e-ticket) and notification on booking status |

> **Rule of thumb:** the entire design hinges on ONE hard problem — **temporarily locking a seat during checkout without permanently blocking it if the user abandons payment.**

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| 💰 **Strong Consistency** | A seat must never be sold to two different people |
| 🔒 **Reliability** | A successful payment must always result in a confirmed booking — never "charged but no seat" |
| ⚡ **Low Latency** | Seat map + hold action must respond quickly, especially during high-demand sales |
| 📈 **Scalability** | Popular routes/sale periods cause many users to compete for the same limited seats simultaneously |
| 🔐 **Security** | Payment details must be handled securely (PCI-compliant flow) |

> Skipped **Offline Support** almost entirely — like Zerodha, booking fundamentally requires a live connection to reserve real inventory; there's nothing meaningful to do offline here.

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

### Extended for Flight Booking specifics

**Flow — Seat Hold → Pay → Confirm (the whole story is one careful sequence)**
```
┌─────────┐ 1. Select seat 12A  ┌─────────────────┐
│ Client  │ ───────────────────▶│ Booking Service    │
└─────────┘                     └────────┬────────┘
                                          │ 2. attempt to hold
                                          ▼    (short TTL lock)
                                 ┌─────────────────┐
                                 │ Seat Inventory        │
                                 │ Service (with a         │
                                 │ distributed lock/         │
                                 │ TTL-based hold per seat)   │
                                 └────────┬────────┘
                              hold failed │  hold succeeded
                          (already taken) │  (locked for e.g. 10 min)
                                 ◀────────┤
                          "seat taken,    │
                           pick another"  ▼
                                 ┌─────────────────┐
                                 │ Client shown a         │
                                 │ countdown timer to        │
                                 │ complete payment            │
                                 └────────┬────────┘
                                          │ 3. pay within hold window
                                          ▼
                                 ┌─────────────────┐
                                 │ Payment Service        │
                                 └────────┬────────┘
                          payment failed  │  payment success
                                 ◀────────┤
                          release the     │
                          seat hold       ▼
                                 ┌─────────────────┐
                                 │ Booking Service          │
                                 │ marks booking CONFIRMED,   │
                                 │ converts hold → permanent   │
                                 │ reservation                    │
                                 └────────┬────────┘
                                          │ 4. notify
                                          ▼
                                 ┌─────────────────┐
                                 │ Notification Service     │
                                 │ (e-ticket, confirmation)    │
                                 └─────────────────┘

If the hold TTL expires (user abandons payment):
Seat Inventory Service auto-releases the seat back to available pool
```

**When you actually draw this in Excalidraw:** draw this as ONE sequential diagram (unlike other topics with 2 separate flows) — the whole point is that hold → pay → confirm is a single tightly-coupled sequence. Highlight the TTL-based hold box; that's the key design decision.

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **UI Layer** | Renders seat map, hold countdown timer, payment screen |
| **ViewModel** | Holds selected seat, hold expiry countdown, booking status |
| **Repository** | Talks to Booking Service, doesn't cache seat availability (must always be live) |
| **Booking Service** | Orchestrates the whole hold → pay → confirm sequence |
| **Seat Inventory Service** | Owns seat availability state, implements the temporary hold/lock mechanism |
| **Payment Service** | Handles secure charging |
| **Notification Service** | Sends confirmation/e-ticket after successful booking |

### B. Data flow — walking through "Booking a Seat"
```
1. User taps seat 12A on the seat map
2. Booking Service asks Seat Inventory Service to place a HOLD
   on 12A — this is a short-lived lock (e.g. TTL = 10 minutes),
   NOT a permanent reservation yet
3. If another user already holds/booked 12A: hold request fails,
   client is told to pick a different seat
4. If hold succeeds: client shows a countdown timer and proceeds
   to the payment screen
5. User completes payment within the hold window
   → Payment Service confirms the charge succeeded
6. Booking Service converts the temporary hold into a permanent
   CONFIRMED booking, Seat Inventory Service marks 12A as sold
7. Notification Service sends the e-ticket/confirmation
8. If the user abandons payment and the hold TTL expires,
   Seat Inventory Service automatically releases 12A back
   to the available pool — no manual cleanup needed
```

### C. Why these choices (tie back to NFRs)
- **TTL-based seat hold (not instant permanent booking)** exists because of **Strong Consistency + UX balance** — it prevents double-booking (only one hold can exist per seat at a time) while still giving the user a few minutes to actually complete payment, rather than requiring an instant all-or-nothing transaction.
- **Auto-release on TTL expiry** exists because of **Scalability** — during a sale, seats abandoned mid-checkout must return to the pool automatically; manual/delayed cleanup would waste inventory and frustrate other buyers.
- **Payment happens AFTER the hold, not before** exists because of **Reliability** — we never charge a user for a seat that isn't provably reserved for them at that moment.
- **Booking Service converts hold → confirmed only after payment success** exists because of **Reliability** — this ordering guarantees "charged but no confirmed seat" can never happen; if payment fails, the hold is simply released, no money was taken.

### D. One trade-off to mention
> "A longer hold window (say 15 min) is more forgiving for users filling in payment details, but ties up inventory longer during high-demand periods, frustrating other buyers watching a seat show as 'unavailable' while someone else may not even complete the purchase. Most systems tune this to a short window (5-10 min) as a deliberate trade-off."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **Implementing the seat hold/lock mechanism at scale** | Redis with a TTL-based key per seat (`SET seat:12A user_id NX EX 600`) — atomic, auto-expiring, no manual cleanup job needed |
| **Two users tap the same seat at the exact same millisecond** | Redis `NX` (set-if-not-exists) is atomic — only one request wins, the other gets an immediate failure |
| **Handling payment gateway timeout (unclear success/fail)** | Idempotent payment confirmation check + reconciliation job to resolve ambiguous states, never blindly retry a charge |
| **Cancellations releasing inventory back** | Cancellation flow reverses the booking, marks seat available again, may trigger a refund flow via Payment Service |

---

## ✅ Recap in One Line
> Hold seat with a short TTL lock (prevents double-booking, auto-releases if abandoned) → pay within the window → only on payment success does the hold become a permanent confirmed booking.

---
