# 🌐 HLD — Design Networking SDK (like Retrofit)

**Examples:** Retrofit, OkHttp, Volley, Ktor Client

> Topic #8 in the HLD practice series. Follows the fixed 4-step framework: FR → NFR → Diagram → Explain.
>
> ⚠️ **Note:** Another **SDK**, but a different shape from Event Logging/Analytics. Those were "collect and ship data out." This one is "take a developer's API definition and turn it into an actual HTTP call, reliably." Think request lifecycle, not event pipeline.

---

## 📋 Step 1: Functional Requirements (FR)

**What can the (developer/host app) DO with this SDK?**

| # | Requirement |
|---|---|
| 1 | Developer can define an API endpoint declaratively (e.g. an interface with annotations) |
| 2 | Developer can make GET/POST/PUT/DELETE calls with path/query/body params |
| 3 | SDK can automatically convert JSON ↔ Kotlin objects (serialization) |
| 4 | Developer can add interceptors (auth headers, logging, retry) |
| 5 | SDK can execute calls synchronously or asynchronously (coroutines/callbacks) |
| 6 | SDK can cache responses and reuse connections |

> **Rule of thumb:** the "user" here is a developer writing `interface ApiService { @GET("/users") suspend fun getUsers(): List<User> }` — think clean API design, not screens.

---

## ⚙️ Step 2: Non-Functional Requirements (NFR)

**How WELL should it work?**

| NFR | Why it matters here |
|---|---|
| ⚡ **Performance** | Reuse connections (keep-alive), avoid redundant DNS/TLS handshakes per call |
| 🔧 **Extensibility** | Easy to plug in custom interceptors, converters, or adapters without touching core |
| 🔒 **Reliability** | Automatic retry on transient failures (timeout, 5xx), no silent failures |
| 🧵 **Thread Safety** | Multiple concurrent calls from different threads must not corrupt shared state |
| 🔋 **Resource Efficiency** | Connection pooling + response caching to save battery/data |

> Skipped **Scalability** in the traditional sense (millions of users) — this SDK runs on ONE device at a time; "scale" here means handling many concurrent calls gracefully within that device.

---

## 🎨 Step 3: HLD Diagram

### Base structure (SDK-style — a request pipeline, not client-server)
```
           Developer defines
           interface + annotations
                    |
                    ↓
           Dynamic Proxy / Code Gen
           (builds the actual call)
                    |
                    ↓
             Request Builder
                    |
                    ↓
           Interceptor Chain
        (auth, logging, retry, cache)
                    |
                    ↓
              OkHttp Core
           (connection pool, TLS)
                    |
                    ↓
                Network
                    |
                    ↓
           Response Converter
           (JSON → Kotlin object)
                    |
                    ↓
           Back to Developer
           (suspend fun result)
```

### Extended internal flow
```
┌───────────────┐ 1. getUsers()   ┌─────────────────┐
│ Developer Code│ ───────────────▶│ Dynamic Proxy      │
│ (interface     │                 │ (Retrofit core)     │
│  method call)   │                └────────┬────────┘
└───────────────┘                          │ 2. build HTTP request
                                            ▼    (method, URL, headers, body)
                                   ┌─────────────────┐
                                   │ Request Builder    │
                                   └────────┬────────┘
                                            │ 3. pass through chain
                                            ▼
                                   ┌─────────────────┐
                                   │ Interceptor Chain   │
                                   │ ┌───────────────┐   │
                                   │ │ Auth Interceptor│   │
                                   │ ├───────────────┤   │
                                   │ │ Cache Interceptor│  │
                                   │ ├───────────────┤   │
                                   │ │ Retry Interceptor│  │
                                   │ ├───────────────┤   │
                                   │ │ Logging Interceptor│ │
                                   │ └───────────────┘   │
                                   └────────┬────────┘
                                            │ 4. execute
                                            ▼
                                   ┌─────────────────┐
                                   │ OkHttp Core         │
                                   │ (connection pool,    │
                                   │  dispatcher, TLS)     │
                                   └────────┬────────┘
                                            │ 5. actual network call
                                            ▼
                                   ┌─────────────────┐
                                   │ Server              │
                                   └────────┬────────┘
                                            │ 6. raw response
                                            ▼
                                   ┌─────────────────┐
                                   │ Response Converter  │
                                   │ (Gson/Moshi/kotlinx) │
                                   └────────┬────────┘
                                            │ 7. typed object
                                            ▼
                                   Back to Developer's
                                   suspend function
```

**When you actually draw this in Excalidraw:** this is a straight pipeline like the Event Logging Library — draw top-to-bottom, numbering 1-7. The one thing worth zooming into separately is the **Interceptor Chain**, since that's the most-asked-about part (draw it as a small stack of boxes, like above).

---

## 🗣️ Step 4: Explain HLD

### A. What each component does

| Component | Job |
|---|---|
| **Dynamic Proxy / Code Gen** | Turns an annotated interface method call into an actual request object (this is Retrofit's core trick — no code you wrote actually implements the interface, it's generated at runtime/compile-time) |
| **Request Builder** | Assembles method, URL, headers, query/path params, and body into an HTTP request |
| **Interceptor Chain** | A pluggable pipeline — each interceptor can inspect/modify the request or response (auth token injection, logging, retry logic, caching) before passing it along |
| **OkHttp Core** | Owns the actual socket-level work — connection pooling, TLS handshake, request dispatch |
| **Response Converter** | Deserializes raw JSON/bytes into the Kotlin data class the developer expects |

### B. Data flow — walking through "Making a Call"
```
1. Developer calls apiService.getUsers() (a suspend function)
2. Dynamic Proxy intercepts this call, builds a Request object
   using the @GET annotation + base URL
3. Request passes through the Interceptor Chain, in order:
   - Auth Interceptor adds "Authorization: Bearer <token>" header
   - Cache Interceptor checks if a cached response is still valid
     → if valid: short-circuit here, return cached response immediately
     → if not: continue
   - Retry Interceptor wraps the call so failures get retried
   - Logging Interceptor logs the outgoing request
4. OkHttp Core picks a pooled connection (or opens a new one) and
   sends the request over the network
5. Response comes back, flows back UP through the same interceptor
   chain (in reverse) — e.g. Cache Interceptor stores it if cacheable
6. Response Converter parses JSON into List<User>
7. The suspend function resumes and returns the typed result to the developer
```

### C. Why these choices (tie back to NFRs)
- **Interceptor Chain (pluggable pipeline)** exists because of **Extensibility** — auth, logging, retry, and caching are all cross-cutting concerns; each one lives as an isolated, addable/removable unit instead of being hardcoded into the core.
- **Connection pooling in OkHttp Core** exists because of **Performance + Resource Efficiency** — reusing an existing TCP/TLS connection for the next request avoids the expensive handshake cost every single time.
- **Cache Interceptor short-circuiting early** exists because of **Performance** — if a valid cached response exists, we skip the network entirely, saving time and battery.
- **Coroutines (suspend functions)** exist because of **Thread Safety** — the caller doesn't manage threads manually; the SDK handles moving work off the main thread and resuming safely.

### D. One trade-off to mention
> "Putting retry logic as an interceptor is simple and composable, but it means retries are somewhat 'invisible' to the developer — they don't see how many attempts happened. An alternative is exposing retry as an explicit parameter per call, giving more control at the cost of more boilerplate for the developer."

---

## 💬 Step 5: Follow-up Discussion (if interviewer digs deeper)

| Area | Quick Answer |
|---|---|
| **How does the dynamic proxy actually work?** | Uses `java.lang.reflect.Proxy` (or KSP/annotation processing) to generate an implementation of the interface at runtime, translating each method call into a request |
| **Handling request cancellation** | Coroutine cancellation propagates down to OkHttp's `Call.cancel()` |
| **Avoiding thundering herd on retry** | Exponential backoff + jitter in the Retry Interceptor |
| **Supporting multiple response formats (JSON, Protobuf)** | Pluggable `Converter.Factory` — same idea as the interceptor pattern, but for serialization |

---

## ✅ Recap in One Line
> Interface call → Dynamic Proxy builds the request → flows through a pluggable Interceptor Chain (auth/cache/retry/log) → OkHttp executes it over a pooled connection → response flows back through converters into a typed object.

---
