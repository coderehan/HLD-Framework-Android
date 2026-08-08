# 📉 HLD — Design Analytics SDK (Android)

**Examples:** Firebase Analytics, Mixpanel, Amplitude, Google Analytics for Apps

> Topic #7 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.
>
> ⚠️ **Note:** Like the Event Logging Library, this is an **SDK**, not a client-server app. The pipeline (queue → batch → persist → upload) is the same underlying mechanism — but an Analytics SDK adds a layer ON TOP: **who** did it (identity), **when in their journey** (session/screen), and **where should it go** (multi-destination). If asked both topics back-to-back in an interview, say this out loud — it shows you see the relationship instead of treating them as unrelated.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the (developer/host app) DO with this SDK?**

| # | Requirement |
|---|---|
| 1 | App can track a custom event (`trackEvent("button_clicked")`) |
| 2 | App can identify a user (`identify(userId)`) and attach traits (name, plan, etc.) |
| 3 | SDK can automatically track screen views (which screen, how long) |
| 4 | SDK can group events into sessions (session start/end, session length) |
| 5 | App can set "super properties" that auto-attach to every event (e.g. app version) |
| 6 | SDK can route the same event to multiple destinations (own backend + Firebase + etc.) |
| 7 | App can respect user consent (opt-out of tracking, GDPR-style) |

> **Rule of thumb:** the FR list is bigger than Event Logging because Analytics SDKs answer "who, when, and where" — not just "what happened."

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| 🔋 **Battery Efficiency** | Same as logging — never wake radio per event, batch uploads |
| 📴 **Reliability (no data loss)** | Events must survive app kill/crash before upload |
| ⚡ **Low Overhead** | `trackEvent()` must return instantly, never block the caller |
| 🔐 **Privacy Compliance** | Must honor opt-out/consent — stop collecting instantly when asked |
| 🔧 **Extensibility** | Adding a new destination (e.g. a new ad network) shouldn't need app code changes |

> Same battery/reliability/overhead NFRs as Event Logging Library — the NEW one here is **Privacy Compliance**, because identity data (user ID, traits) makes this legally sensitive in a way raw anonymous events aren't.

---

## 🎨 Step 3: HLD Diagram

### Base structure (SDK-style — no "User → UI", it's "Host App → SDK")
```
              Host App
                 |
                 ↓
   trackEvent() / identify() / screen()   ← public API, returns instantly
                 |
                 ↓
          Consent Check
                 |
                 ↓
        Session & Identity
        Enrichment Layer
                 |
                 ↓
           Event Queue
                 |
                 ↓
         Batching Processor
                 |
                 ↓
        Local Persistence
                 |
                 ↓
       Destination Router
        /        |        \
   Own Backend  Firebase  Ad Network SDK
```

### Extended internal flow
```
┌──────────┐ 1. trackEvent()  ┌─────────────────┐
│ Host App │ ────────────────▶│ Public API        │
└──────────┘                  └────────┬────────┘
                                        │ 2. check consent
                                        ▼
                               ┌─────────────────┐
                               │ Consent Manager    │
                               └────────┬────────┘
                            opted-out → │ → drop event, do nothing
                            opted-in  → │
                                        ▼
                               ┌─────────────────┐
                               │ Enrichment Layer   │
                               │ (adds: userId,      │
                               │  sessionId, screen,   │
                               │  super properties)     │
                               └────────┬────────┘
                                        │ 3. enqueue (non-blocking)
                                        ▼
                               ┌─────────────────┐
                               │ In-Memory Queue     │
                               └────────┬────────┘
                                        │ 4. batch
                                        ▼
                               ┌─────────────────┐
                               │ Local DB (buffer)   │
                               └────────┬────────┘
                                        │ 5. scheduled upload
                                        ▼
                               ┌─────────────────┐
                               │ Destination Router  │
                               └───┬─────────┬─────┘
                                   ▼         ▼
                          ┌──────────┐  ┌──────────┐
                          │ Own       │  │ 3rd-party  │
                          │ Backend   │  │ SDKs        │
                          └──────────┘  └──────────┘
```

**When you actually draw this in Excalidraw:** draw it as one pipeline like Event Logging, but call out TWO new boxes clearly: **Consent Manager** (early, acts as a gate) and **Destination Router** (late, fans out to multiple places). Those two are what make this topic distinct.

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **Public API** | `trackEvent()`, `identify()`, `screen()` — simple, thread-safe, instant |
| **Consent Manager** | Gatekeeper — if user opted out, drop the event right here, before anything else happens |
| **Enrichment Layer** | Auto-attaches identity (userId), session info, current screen, and super properties |
| **In-Memory Queue → Batching Processor → Local DB** | Same reliable pipeline as the Event Logging Library |
| **Destination Router** | Decides which backend(s)/SDK(s) each event should be forwarded to |

### B. Data flow — walking through "Tracking an Event"
```
1. Host app calls identify("user_123") once at login
2. Later, host app calls trackEvent("checkout_completed", {amount: 999})
3. Consent Manager checks: has this user opted in? → yes, continue
4. Enrichment Layer attaches: userId=user_123, sessionId, current screen,
   timestamp, super properties (app version, etc.)
5. Event pushed to in-memory queue → batched → persisted to Local DB
   (same reliable pipeline as Event Logging Library)
6. On upload, Destination Router sends the batch to:
   - Own backend (for internal dashboards)
   - Firebase (if configured)
   - Any other registered destination
```

### C. Data flow — walking through "User Opts Out"
```
1. User taps "Do not track me" in app settings
2. App calls sdk.optOut()
3. Consent Manager flag flips instantly — ALL future events are dropped
   at step 3 above, before enrichment even happens
4. (Depending on legal requirement) SDK may also trigger a delete-request
   to the backend for previously collected data
```

### D. Why these choices (tie back to NFRs)
- **Consent Manager sits FIRST, before enrichment** exists because of **Privacy Compliance** — we shouldn't even attach identity/session data to an event if the user opted out; dropping early means zero risk of leaking anything downstream.
- **Destination Router as a separate late stage** exists because of **Extensibility** — adding a new ad network destination means writing one new router entry, not touching the queue/batch/persist pipeline at all.
- **Enrichment happens once, centrally** exists because of **Low Overhead** — the host app doesn't need to manually attach userId/session to every single `trackEvent()` call; the SDK does it once, consistently.

### E. One trade-off to mention
> "Routing every event to multiple destinations at once increases upload payload/battery cost per batch. An alternative is destination-specific sampling — send 100% of events to your own backend, but only 10% to third-party SDKs — trading data completeness for efficiency where it matters less."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **How is this different from a generic event logger?** | Logger just ships raw events; Analytics SDK adds identity, sessions, consent, and multi-destination routing on top |
| **Handling GDPR "right to be forgotten"** | SDK triggers a delete API call to backend with the userId; backend purges historical events |
| **Session boundary definition** | New session starts if app was backgrounded > N minutes (e.g. 30 min) |
| **Avoiding event loss during opt-out toggle race** | Consent check happens synchronously before enrichment — no event can slip through mid-toggle |

---

## ✅ Recap in One Line
> Same reliable pipeline as an Event Logger (queue → batch → persist → upload), but wrapped with a Consent gate up front and a Destination Router at the end — because analytics data is about identity and needs to go multiple places, safely.

---
