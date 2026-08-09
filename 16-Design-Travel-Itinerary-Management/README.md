# 🗺️ HLD — Design Travel Itinerary Management (Android)

**Examples:** TripIt, Google Trips (discontinued but a great reference), Wanderlog

> Topic #16 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.
>
> ⚠️ **Note:** Unlike the booking-focused topics, this app doesn't sell anything — it **aggregates** bookings the user already made elsewhere (flights, hotels, car rentals — often from different apps/emails) into one unified timeline. The core challenge: parsing/normalizing messy, inconsistent data from many sources into one clean structure.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the user DO?**

| # | Requirement |
|---|---|
| 1 | User can forward a booking confirmation email to auto-add it to their itinerary |
| 2 | User can manually add a trip item (flight, hotel, activity) |
| 3 | User can view a chronological timeline for a trip, merging all items |
| 4 | User can view trip details offline (useful while traveling, often with no signal) |
| 5 | User can share an itinerary with a travel companion |
| 6 | System can alert on schedule conflicts (e.g. two overlapping hotel bookings) |

> **Rule of thumb:** the "input" side (parsing a confirmation email) is arguably harder than the "output" side (showing a timeline) — spend real time on this in the interview.

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| 📴 **Offline-First** | Travelers often have no signal (flights, remote areas) — the itinerary MUST work offline |
| 🔧 **Extensibility** | Parsing must handle many booking source formats (airlines, hotel chains, car rentals) without a rewrite each time |
| 🎯 **Parsing Accuracy** | Wrong date/time extracted from an email is worse than not parsing it at all |
| 🔁 **Data Consistency** | Same trip shared across devices/companions should show the same data |
| ⚡ **Low Latency** | Timeline view must render instantly — this is what users check constantly while traveling |

> Skipped **Scalability** as a top concern — per-user trip data is small; this isn't a shared-inventory or high-throughput problem like flight booking.

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

> **Key point:** Room here is not just a cache — for this app, the **local DB is arguably the primary experience**, since offline access while traveling is a core requirement, not an edge case.

### Extended for Itinerary Management specifics

**Flow 1 — Auto-Adding via Forwarded Email**
```
┌─────────┐  1. forward email    ┌─────────────────┐
│ User's  │ ────────────────────▶│ Email Ingestion     │
│ Email   │                       │ Service               │
└─────────┘                       └────────┬────────┘
                                            │ 2. classify email
                                            ▼    (flight? hotel? car?)
                                   ┌─────────────────┐
                                   │ Parser Router          │
                                   │ (picks the right          │
                                   │  parser for this source)    │
                                   └────────┬────────┘
                                            │ 3. extract structured
                                            ▼    data
                                   ┌─────────────────┐
                                   │ Booking-Specific        │
                                   │ Parser (Airline parser,   │
                                   │ Hotel parser, etc.)          │
                                   └────────┬────────┘
                                            │ 4. normalize into
                                            ▼    common Trip Item model
                                   ┌─────────────────┐
                                   │ Trip Item DB             │
                                   └────────┬────────┘
                                            │ 5. sync to device
                                            ▼
                                   Client's Room DB
                                   (available offline)
```

**Flow 2 — Viewing the Timeline**
```
┌─────────┐  1. Open trip        ┌─────────────────┐
│ Client  │ ────────────────────▶│ Local Trip Store    │
└─────────┘                       │ (Room — reads FIRST,  │
                                   │  works with no network) │
                                   └────────┬────────┘
                                            │ 2. sort all items
                                            ▼    chronologically
                                   ┌─────────────────┐
                                   │ Timeline Builder         │
                                   │ (merges flights, hotels,   │
                                   │  activities into one         │
                                   │  ordered view)                 │
                                   └─────────────────┘
```

**When you actually draw this in Excalidraw:** draw "Email Ingestion" and "Viewing Timeline" separately. The **Parser Router → Booking-Specific Parser** pattern is worth zooming into — it's the plugin/strategy pattern applied to email parsing.

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **UI Layer** | Renders trip timeline, item details, conflict warnings |
| **ViewModel** | Holds current trip's timeline state |
| **Repository** | Always reads from Room first (offline-first, same pattern as Cart & Wishlist) |
| **Room (Local DB)** | Primary store for trip data — works fully offline |
| **Email Ingestion Service** | Receives forwarded confirmation emails |
| **Parser Router** | Classifies the email type and routes it to the right parser |
| **Booking-Specific Parsers** | Each knows how to extract structured data from ONE source's email format (e.g. a dedicated "United Airlines parser," "Marriott parser") |
| **Trip Item DB** | Backend source of truth for all normalized trip items |
| **Timeline Builder** | Merges different item types (flight/hotel/activity) into one chronological view |

### B. Data flow — walking through "Auto-Adding a Booking"
```
1. User forwards a hotel confirmation email to a special
   trips@ import address
2. Email Ingestion Service receives it, Parser Router inspects
   the sender/content to classify it as "hotel booking"
3. The Hotel-specific Parser extracts check-in/out dates,
   address, confirmation number from the email body
4. Extracted data is normalized into a common Trip Item model
   (same shape regardless of source: type, start time, end time,
   location, details) — this common model is what makes merging
   different booking types into one timeline possible
5. Saved to Trip Item DB, then synced down to the client's Room DB
6. Timeline Builder re-sorts and the new hotel stay appears in
   the trip's timeline, correctly ordered against existing flights
```

### C. Data flow — walking through "Viewing Offline"
```
1. User opens the trip while on a flight with no signal
2. Repository reads directly from Room — no network call is
   even attempted for the initial render
3. Timeline Builder merges all locally-stored items into one
   sorted view
4. If the user makes edits offline (e.g. adds a manual activity),
   it's queued for sync (same pattern as Cart & Wishlist's
   Sync Queue) and pushed once connectivity returns
```

### D. Why these choices (tie back to NFRs)
- **Room as the primary read path (not a cache-on-top-of-network)** exists because of **Offline-First** — this app's core use case (checking your itinerary while traveling) often has zero connectivity; the local DB isn't an optimization here, it's the actual product requirement.
- **Parser Router + per-source parsers (plugin pattern)** exists because of **Extensibility** — adding support for a new airline/hotel chain's email format means writing one new parser, without touching the ingestion pipeline or the common Trip Item model.
- **Normalizing into one common Trip Item model** exists because of **Low Latency + simplicity** — the Timeline Builder only ever needs to understand ONE shape of data, regardless of how many wildly different source formats feed into it.

### E. One trade-off to mention
> "Auto-parsing emails is convenient but inherently error-prone — a confirmation email format changing slightly can silently break a parser. A safer design always shows the user a 'review before adding' confirmation screen after parsing, rather than trusting the extraction blindly and adding straight to the timeline."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **Handling a parser failing on a new email format** | Fallback: flag as "unrecognized," let user manually fill in details, log the failure for the parser to be improved later |
| **Detecting schedule conflicts** | Timeline Builder checks for overlapping time ranges across items during merge, flags conflicts in UI |
| **Syncing edits made on 2 devices** | Same offline-first sync pattern as Cart & Wishlist — local write first, queued sync, last-write-wins or field-level merge |
| **Supporting many booking types (car rental, train, restaurant reservation)** | Just another parser + the common Trip Item model already supports arbitrary item types |

---

## ✅ Recap in One Line
> Local DB (Room) is the primary experience, not a cache (offline-first by design) → forwarded emails get classified and routed to a source-specific parser → normalized into one common Trip Item model → Timeline Builder merges everything into one clean, chronological view.

---
