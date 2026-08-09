# 🏨 HLD — Design Hotel Booking System (Android)

**Examples:** OYO, Booking.com, MakeMyTrip Hotels, Airbnb

> Topic #17 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.
>
> ⚠️ **Note:** Very similar core problem to the Flight Booking System — reserve limited inventory correctly under concurrent demand. The key DIFFERENCE: a flight seat is a single fixed unit, but a hotel room is booked for a **date RANGE**, and a property has **multiple room types with different counts** — that range-based inventory is what makes this topic distinct.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the user DO?**

| # | Requirement |
|---|---|
| 1 | User can search hotels by location, check-in/check-out dates, guests |
| 2 | User can view room types and availability for those dates |
| 3 | User can hold a room temporarily while completing payment |
| 4 | User can pay and confirm the booking |
| 5 | User can view/cancel an existing booking |
| 6 | System can prevent overbooking a room type beyond its available count for any overlapping date range |

> **Rule of thumb:** say this out loud early — "a room's availability isn't a single yes/no, it's a per-date count that must stay non-negative across the whole stay range." That's the crux of the problem.

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| 💰 **Strong Consistency** | A room type must never be overbooked beyond its actual count for any date |
| 🔒 **Reliability** | A successful payment must always result in a confirmed booking |
| ⚡ **Low Latency** | Availability search across date ranges must be fast, even for popular destinations |
| 📈 **Scalability** | Weekend/holiday searches spike traffic on popular properties |
| 🔐 **Security** | Payment handling needs bank-grade protection |

> Same NFR shape as Flight Booking (unsurprising, since it's the same class of problem) — the implementation detail that differs is HOW inventory is tracked (per-date-range vs per-seat).

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

### Extended for Hotel Booking specifics

**Flow — Hold → Pay → Confirm (same shape as Flight Booking, different inventory model)**
```
┌─────────┐ 1. Select room type,  ┌─────────────────┐
│ Client  │    dates (3 nights)     │ Booking Service     │
└─────────┘ ─────────────────────▶ └────────┬────────┘
                                             │ 2. attempt to hold
                                             ▼    ONE unit of this room
                                    ┌─────────────────┐    type across ALL
                                    │ Room Inventory        │    3 nights
                                    │ Service (per-date,      │
                                    │  per-room-type counter)  │
                                    └────────┬────────┘
                                 hold failed │  hold succeeded
                            (0 available on  │  (decremented for
                             at least 1 of   │   each of the 3 dates,
                             the 3 nights)   │   short TTL)
                                    ◀────────┤
                             "not available, │
                              try other dates"▼
                                    ┌─────────────────┐
                                    │ Client shown countdown │
                                    │ to complete payment       │
                                    └────────┬────────┘
                                             │ 3. pay within hold window
                                             ▼
                                    ┌─────────────────┐
                                    │ Payment Service        │
                                    └────────┬────────┘
                             payment failed  │  payment success
                                    ◀────────┤
                             release the     │
                             held inventory  ▼
                                    ┌─────────────────┐
                                    │ Booking Service          │
                                    │ marks booking CONFIRMED    │
                                    └────────┬────────┘
                                             │ 4. notify
                                             ▼
                                    Notification Service
```

**When you actually draw this in Excalidraw:** draw the sequence like above (very close to Flight Booking's diagram), but add a small side-diagram showing the **per-date inventory counter concept**:
```
Room Type: Deluxe (10 total rooms)

  Date:        Aug 10   Aug 11   Aug 12
  Available:     7        6        8

A 3-night booking (Aug 10-12) needs 1 unit available
on ALL THREE dates — not just one.
```

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **UI Layer** | Renders search results, room type selection, calendar, payment screen |
| **ViewModel** | Holds selected dates, room type, hold countdown |
| **Repository** | Talks to Booking Service, doesn't cache live availability |
| **Booking Service** | Orchestrates the hold → pay → confirm sequence |
| **Room Inventory Service** | Owns per-date, per-room-type availability counters and the hold mechanism |
| **Payment Service** | Handles secure charging |
| **Notification Service** | Sends confirmation after successful booking |

### B. Data flow — walking through "Booking a Room"
```
1. User searches "Deluxe Room, Aug 10-12, 1 guest" at a hotel
2. Booking Service asks Room Inventory Service to HOLD 1 unit
   of "Deluxe Room" for EVERY date in the range (Aug 10 AND Aug 11
   — checkout day, Aug 12, doesn't need a hold since the guest
   leaves that morning)
3. Room Inventory Service checks: is availability > 0 for
   Deluxe Room on BOTH Aug 10 and Aug 11?
   → if any date in the range has 0 available: hold fails entirely
   → if all dates have availability: decrement the counter for
     each date, place a short TTL hold
4. Client shows countdown, user proceeds to payment
5. On payment success: Booking Service confirms the booking,
   the temporary hold becomes a permanent decrement
6. On payment failure or TTL expiry: the hold is released,
   counters for each date are incremented back
```

### C. Why these choices (tie back to NFRs)
- **Per-date, per-room-type counters (not a single boolean)** exists because of **Strong Consistency** — a hotel has multiple identical rooms of the same type; tracking a COUNT per date (not a single lock) correctly reflects that N rooms of the same type can be booked by N different guests, as long as they don't collectively exceed the count on any shared date.
- **Hold must succeed across the ENTIRE date range atomically** exists because of **Strong Consistency** — holding Aug 10 but failing Aug 11 halfway through would leave inconsistent partial holds; the whole range must be checked and held (or rejected) as one atomic operation.
- **Same TTL-hold-then-confirm pattern as Flight Booking** exists because it's fundamentally the same class of problem (reserve first, confirm after payment) — reusing the pattern is a strength, not a shortcut; the interviewer wants to see you recognize the shared shape.

### D. One trade-off to mention
> "Checking and holding availability across a full date range in one atomic step is more complex than a single-seat lock, but it's non-negotiable — a naive per-date-independent check (checking each date separately, without a shared lock) could let two different bookings both succeed for overlapping ranges due to a race condition between the checks."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **Implementing the atomic multi-date hold** | A Redis Lua script (or DB transaction) that checks-and-decrements all date counters for the range in one atomic operation — prevents race conditions between check and decrement |
| **Long stays (e.g. 30 nights) — is checking 30 date counters slow?** | Store availability in date-range-friendly structures (e.g. a segment tree or precomputed daily counts) rather than looping row-by-row on every check |
| **Handling group bookings (5 rooms at once)** | Same atomic hold logic, just decrementing by 5 instead of 1, still across the full date range |
| **Cancellations near check-in date** | Cancellation policy logic (refund %) sits in Booking Service; inventory is released back regardless once cancelled |

---

## ✅ Recap in One Line
> Same hold → pay → confirm pattern as Flight Booking, but inventory is tracked as a per-date, per-room-type counter — and a hold must atomically succeed across the ENTIRE stay's date range, or it fails entirely.

---
