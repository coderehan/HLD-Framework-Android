# 💵 HLD — Design Expense Management System (Android)

**Examples:** Splitwise, Google Pay group expenses, Tricount

> Topic #14 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.
>
> ⚠️ **Note:** The core problem here isn't UI, it's a **graph/math problem**: given who-owes-whom across many expenses, compute the minimum set of settlements. That's what interviewers actually want to see you reason through.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the user DO?**

| # | Requirement |
|---|---|
| 1 | User can create a group (e.g. "Goa Trip") and add members |
| 2 | User can add an expense, split equally/unequally/by percentage among members |
| 3 | User can view "who owes whom, how much" per group |
| 4 | User can settle up (mark a debt as paid) |
| 5 | System can simplify debts (minimize number of transactions needed) |
| 6 | User can get notified when added to a group or an expense involving them |

> **Rule of thumb:** an expense isn't just "$100 split 4 ways" — it creates multiple pairwise debts. The system's real job is tracking and simplifying a web of these debts.

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| 💰 **Correctness** | Money math must be exact — no rounding errors that leave a few cents unaccounted for |
| 🔒 **Consistency** | Two members shouldn't see different balances for the same group |
| ⚡ **Low Latency** | Adding an expense and seeing updated balances should feel instant |
| 📴 **Offline Support** | Adding an expense on a trip with no signal should still work, sync later |
| 🔧 **Auditability** | Every balance must be traceable back to the expenses that created it (for disputes) |

> Skipped **Scalability** as a top concern — group sizes are small (rarely more than 20-30 people), so this isn't a "millions of concurrent users on one entity" problem like a social feed.

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

### Extended for Expense Management specifics

**Flow 1 — Adding an Expense**
```
┌─────────┐  1. Add expense    ┌─────────────────┐
│ Client  │    ($100, split      │ Expense Service    │
│         │    4 ways)            │                      │
│         │ ─────────────────────▶└────────┬────────┘
└─────────┘                               │ 2. save expense +
                                           │    compute pairwise splits
                                           ▼
                                  ┌─────────────────┐
                                  │ Expense DB           │
                                  │ (raw expense record)  │
                                  └────────┬────────┘
                                           │ 3. update running balances
                                           ▼
                                  ┌─────────────────┐
                                  │ Balance Ledger        │
                                  │ (who-owes-whom per     │
                                  │  group, incrementally    │
                                  │  updated)                  │
                                  └─────────────────┘
```

**Flow 2 — Viewing Simplified Balances**
```
┌─────────┐  1. View group      ┌─────────────────┐
│ Client  │    balances           │ Balance Service     │
│         │ ─────────────────────▶└────────┬────────┘
└─────────┘                               │ 2. read raw pairwise
                                           ▼    balances from ledger
                                  ┌─────────────────┐
                                  │ Debt Simplification    │
                                  │ Algorithm (graph-based)  │
                                  └────────┬────────┘
                                           │ 3. minimal set of
                                           ▼    settlement transactions
                                  Return to client:
                                  "Rehan owes Priya ₹500"
                                  (not 5 separate smaller debts)
```

**When you actually draw this in Excalidraw:** draw "Add Expense" and "View Balances" separately, but the real diagram worth spending time on is a **small graph diagram** showing 3-4 people with debt arrows between them, then simplified — that visual sells the "debt simplification" concept instantly.

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **UI Layer** | Renders group screen, add-expense form, balance summary |
| **ViewModel** | Holds group state, expense list, computed balances |
| **Repository** | Caches group/expense data locally, syncs with backend |
| **Room (Local DB)** | Stores groups/expenses for offline access and adding expenses offline |
| **Expense Service** | Validates and stores new expense records |
| **Balance Ledger** | Maintains running pairwise balances per group (incrementally updated, not recomputed from scratch each time) |
| **Debt Simplification Algorithm** | Reduces a tangled web of pairwise debts into the minimum number of actual payments needed |

### B. Data flow — walking through "Adding an Expense"
```
1. User adds a ₹1000 dinner expense in "Goa Trip" group, split
   equally among 4 people (₹250 each)
2. Expense Service saves the raw expense record (who paid, how
   it was split) to Expense DB — this is the source of truth,
   never deleted, only added to (for auditability)
3. Balance Ledger is updated incrementally: the 3 people who
   didn't pay each now owe the payer ₹250 more
4. Updated balances are pushed back to the client, UI reflects
   the new "you owe" / "you're owed" amounts instantly
```

### C. Data flow — walking through "Debt Simplification"
```
Example: in a group of 3 —
  A owes B ₹500
  B owes C ₹500
  C owes A ₹200

Naive view: 3 separate transactions needed.

Debt Simplification Algorithm treats this as a graph problem:
1. Compute each person's NET balance:
   A: -500 (owes) + 200 (owed) = -300 net
   B: +500 (owed) - 500 (owes) = 0 net
   C: +500 (owed) - 200 (owes) = +300 net
2. B nets to zero — drops out of settlement entirely
3. Only A and C have non-zero net balances
4. Simplified result: "A pays C ₹300" — ONE transaction
   instead of three
```

### D. Why these choices (tie back to NFRs)
- **Balance Ledger updated incrementally (not recomputed from all expenses every time)** exists because of **Low Latency** — with hundreds of expenses in a long trip, recalculating from scratch on every view would get slow; incremental updates keep reads fast.
- **Raw Expense DB kept as immutable source of truth** exists because of **Auditability** — if a balance is ever disputed, you can trace back to exactly which expenses produced it, rather than trusting a black-box number.
- **Debt Simplification via net balances (graph reduction)** exists because of **UX + Correctness** — showing users the minimum number of actual payments needed is both simpler to understand and reduces real-world transaction friction (fewer UPI transfers needed).
- **Offline-first (Room writes first, like Cart)** exists because of **Offline Support** — adding a dinner expense on a trip with no signal must work immediately, syncing once back online (same local-first pattern as the Cart & Wishlist topic).

### E. One trade-off to mention
> "Fully simplifying debts across an entire group is elegant but can feel confusing — e.g. 'why am I paying someone I never directly owed?' A common middle ground is offering BOTH views: raw pairwise debts (transparent) and a 'simplify' button (convenient), letting the user choose."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **Handling rounding errors in unequal splits** | Distribute the leftover paise/cents to the first N people, tracked explicitly, never silently dropped |
| **Currency conversion for multi-currency trips** | Store expense in original currency + a converted amount at time of entry (exchange rate can drift, don't recompute later) |
| **Concurrent edits (2 people edit same expense)** | Optimistic concurrency — version/timestamp check, last-write-wins with a conflict warning shown |
| **Settling up — marking a debt as paid** | Creates a special "settlement" expense record that offsets the balance, keeping the audit trail intact |

---

## ✅ Recap in One Line
> Every expense is stored immutably (auditability) → Balance Ledger updates incrementally (speed) → Debt Simplification reduces the group's debt graph to net balances, minimizing actual payments needed.

---
