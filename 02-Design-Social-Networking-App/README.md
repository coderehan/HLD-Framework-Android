# 🌐 HLD — Design Social Networking App (Android)

**Examples:** Facebook, Instagram, Twitter/X, LinkedIn

> Topic #2 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the user DO?**

| # | Requirement |
|---|---|
| 1 | User can create a post (text/image/video) |
| 2 | User can view a personalized feed of posts |
| 3 | User can like, comment, and share posts |
| 4 | User can follow/unfollow other users |
| 5 | User can get notified on new activity (likes, comments, follows) |
| 6 | User can view someone's profile with their posts |

> **Rule of thumb:** the "feed" is the heart of this app — most of the design effort goes there.

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| 📈 **Scalability** | Millions of users posting/scrolling at once — feed generation must scale |
| ⚡ **Low Latency** | Feed should load and scroll smoothly, likes/comments feel instant |
| 🔔 **Real-time Updates** | Notifications (new like/comment/follow) should arrive quickly |
| 📴 **Offline Support** | Last-loaded feed should still be viewable without network |
| 🔋 **Battery Efficiency** | Background feed refresh & notifications shouldn't drain battery |

> Skipped strict **Consistency** here — unlike payments, a slightly-delayed like count isn't critical (eventual consistency is fine).

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

### Extended for Social Networking specifics

**Flow 1 — Creating a Post**
```
┌─────────┐  1. Create post   ┌────────────────┐
│ Client  │ ────────────────▶ │ Post Service    │
└─────────┘                   └────────┬────────┘
                                        │ 2. Save post
                                        ▼
                               ┌─────────────────┐
                               │ Post DB          │
                               └────────┬────────┘
                                        │ 3. If media attached
                                        ▼
                               ┌─────────────────┐
                               │ Media/CDN Storage │
                               └────────┬────────┘
                                        │ 4. Notify followers
                                        ▼
                               ┌─────────────────┐
                               │ Feed Fan-out      │
                               │ Service           │
                               └─────────────────┘
```

**Flow 2 — Viewing the Feed**
```
┌─────────┐  1. Request feed   ┌─────────────────┐
│ Client  │ ─────────────────▶ │ Feed Service     │
└─────────┘                    └────────┬────────┘
                                         │ 2. Fetch pre-computed
                                         ▼    feed OR compute on-the-fly
                                ┌─────────────────┐
                                │ Feed Cache        │
                                │ (Redis)           │
                                └────────┬────────┘
                                         │ 3. Fetch post + media details
                                         ▼
                                ┌─────────────────┐
                                │ Post DB + CDN      │
                                └─────────────────┘
```

**When you actually draw this in Excalidraw:** draw "Create Post" and "View Feed" as two separate small diagrams, just like the e-commerce "Browse" vs "Checkout" split. Keeps each explanation focused.

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **UI Layer** | Renders feed, post composer, profile screens |
| **ViewModel** | Holds feed state, pagination state, survives rotation |
| **Repository** | Decides: show cached feed instantly or fetch fresh |
| **Room (Local DB)** | Stores last-loaded feed for offline viewing |
| **Post Service** | Handles creating/editing/deleting posts |
| **Feed Fan-out Service** | Pushes a new post out to followers' feeds |
| **Feed Cache (Redis)** | Stores a pre-built feed per user so it loads instantly |
| **Media/CDN Storage** | Stores and serves images/videos fast, close to the user |
| **Notification Service** | Pushes real-time alerts for likes/comments/follows |

### B. Data flow — walking through "Create Post"
```
1. User writes a post and taps "Share"
2. UI sends it to ViewModel → Repository → Post Service
3. Post Service saves post to Post DB
4. If there's an image/video, it's uploaded to CDN storage
5. Feed Fan-out Service pushes this new post into the cached
   feed of every follower (so it shows up next time they open the app)
```

### C. Data flow — walking through "View Feed"
```
1. User opens the app / pulls to refresh
2. UI asks ViewModel → Repository for feed
3. Repository shows Room's cached feed instantly (feels fast)
4. In background, Repository calls Feed Service
5. Feed Service checks Redis for a pre-built feed for this user
   → hit: return immediately
   → miss: build it on-the-fly from Post DB, then cache it
6. New feed replaces the old one in Room + UI updates
```

### D. Why these choices (tie back to NFRs)
- **Feed Fan-out on write** exists because of **Low Latency** — pre-building each follower's feed when a post is created means reading the feed later is instant, instead of computing it fresh every time someone opens the app.
- **Redis feed cache** exists because of **Scalability** — millions of feed reads per minute would crush the DB otherwise.
- **Room local cache** exists because of **Offline Support** — last feed stays visible even with no network.

### E. One trade-off to mention
> "Fan-out on write (pushing posts to every follower's cached feed immediately) is fast to read but expensive for users with millions of followers (celebrities). A common fix is a hybrid: fan-out for normal users, but fan-out-on-read (compute on demand) for celebrity accounts."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **Celebrity/viral post problem** | Hybrid fan-out: pre-push for normal users, compute-on-read for huge accounts |
| **Feed ranking (not just chronological)** | Add a Ranking Service that scores posts by engagement/recency before caching |
| **Real-time notifications** | WebSocket/Push service subscribed to a notification queue |
| **Duplicate/spam post prevention** | Rate limiting at Post Service + content moderation check before save |

---

## ✅ Recap in One Line
> Create Post = save + fan-out to followers' cached feeds (write-heavy, optimized for fast reads later). View Feed = Room cache → Redis cache → DB (speed at every layer).

---
