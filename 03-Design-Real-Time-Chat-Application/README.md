# 💬 HLD — Design Real-Time Chat Application (Android)

**Examples:** WhatsApp, Telegram, Messenger, Slack

> Topic #3 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the user DO?**

| # | Requirement |
|---|---|
| 1 | User can send/receive 1-to-1 messages in real-time |
| 2 | User can see message status (sent / delivered / read) |
| 3 | User can view chat history (persisted, scrollable) |
| 4 | User can send messages while offline (queued, sent when back online) |
| 5 | User can see when the other person is typing / online status |
| 6 | User can get push notifications for new messages |

> **Rule of thumb:** group chat is usually a follow-up extension, not the core — start with 1-to-1.

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| ⚡ **Low Latency** | Messages must feel instant — this is the whole point of "real-time" |
| 📴 **Offline Support** | Compose/queue messages with no network, auto-send once reconnected |
| 🔒 **Reliability** | A message must never silently get lost |
| 📈 **Scalability** | Millions of concurrent open connections (one per active user) |
| 🔐 **Security** | Messages should be encrypted end-to-end or in transit at minimum |

> Skipped heavy **Consistency** — unlike payments, slight delivery-order variance is acceptable as long as nothing is lost.

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
           Room (Local)   WebSocket (Network)
                     |
                     ↓
                  Backend
```

> **Key difference from other apps:** Retrofit (request/response) alone isn't enough here — we need a **persistent connection** (WebSocket) so the server can *push* messages to the client instantly, instead of the client repeatedly asking "any new messages?"

### Extended for Real-Time Chat specifics

**Flow 1 — Sending a Message**
```
┌─────────┐  1. Save locally    ┌──────────────┐
│ Sender  │ ──(status: SENDING)─▶│  Room DB      │
│ Client  │                      │  (local copy) │
└────┬────┘                      └──────────────┘
     │ 2. Send over WebSocket
     ▼
┌─────────────────┐   3. Persist    ┌───────────────┐
│ Chat Gateway      │ ──────────────▶│ Message DB     │
│ (WebSocket Server) │                └───────────────┘
└────────┬───────────┘
         │ 4. Is receiver online?
         ▼
   ┌─────────────┐  yes  ┌──────────────────┐
   │ Presence     │──────▶│ Push over          │
   │ Service      │       │ WebSocket to        │
   └─────────────┘       │ Receiver Client      │
         │ no                         
         ▼                            
   ┌─────────────┐                    
   │ Push          │ ── deliver as OS notification
   │ Notification  │
   │ Service       │
   └─────────────┘
```

**Flow 2 — Message Status Update (Sent → Delivered → Read)**
```
Receiver's client ACKs message
        │
        ▼
Chat Gateway updates Message DB status
        │
        ▼
Chat Gateway pushes status update back to Sender over WebSocket
        │
        ▼
Sender's UI updates tick marks (✓ → ✓✓ → ✓✓ blue)
```

**When you actually draw this in Excalidraw:** draw "Send Message" and "Status Update" as two separate small diagrams, same pattern as before.

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **UI Layer** | Renders chat bubbles, typing indicator, input box |
| **ViewModel** | Holds chat screen state, message list, connection status |
| **Repository** | Decides: read from Room (history) or listen to WebSocket (live) |
| **Room (Local DB)** | Stores full chat history + a local outbox for offline-queued messages |
| **Chat Gateway (WebSocket Server)** | Keeps a persistent open connection per user; routes messages |
| **Presence Service** | Tracks who's online/offline/typing right now |
| **Message DB** | Source of truth for all messages + their delivery status |
| **Push Notification Service** | Wakes up a receiver who's offline/app-closed |

### B. Data flow — walking through "Send Message"
```
1. User types and hits send
2. Message is instantly saved in Room with status = SENDING (UI shows it right away)
3. Repository sends it over the open WebSocket connection to Chat Gateway
4. Chat Gateway persists it in Message DB, marks status = SENT
5. Chat Gateway checks Presence Service:
   → receiver online: push message instantly over their WebSocket connection
   → receiver offline: trigger Push Notification Service instead
6. Once delivered, status flows back to sender → Room updated → UI tick changes
```

### C. Data flow — walking through "Offline Send"
```
1. User sends a message with no network
2. Message saved in Room with status = QUEUED (shown as a clock icon)
3. WebSocket connection drops → app listens for reconnect
4. On reconnect, Repository resends everything in the QUEUED outbox
5. Once Chat Gateway acknowledges, status updates to SENT
```

### D. Why these choices (tie back to NFRs)
- **WebSocket instead of Retrofit polling** exists because of **Low Latency** — a persistent connection lets the server push instantly, instead of the client asking "anything new?" every few seconds (which is both slow and battery-draining).
- **Room outbox queue** exists because of **Offline Support + Reliability** — a message is never lost even if sent mid-tunnel with no signal; it just waits and retries.
- **Presence Service check before delivery** exists because of **Scalability** — no point trying to push over a WebSocket connection that's already closed; fall back to push notification instead.

### E. One trade-off to mention
> "Keeping millions of WebSocket connections open 24/7 is expensive for the backend. An alternative is long-polling, which is simpler to scale but adds latency — WebSocket wins here specifically because our top NFR is real-time feel."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **Scaling millions of WebSocket connections** | Horizontally scale Chat Gateway servers behind a load balancer, with sticky sessions or a shared connection registry |
| **Group chat support** | Chat Gateway fans out one message to all group members' open connections, same idea as social feed fan-out |
| **End-to-end encryption** | Encrypt on sender device, only decrypt on receiver device — server just relays encrypted blobs |
| **Message ordering across devices** | Use a server-assigned sequence number/timestamp per chat, not device local time |

---

## ✅ Recap in One Line
> Send = save locally first (instant feel) → push over WebSocket → persist → deliver live or notify. Offline = queue in Room, auto-retry on reconnect.

---
