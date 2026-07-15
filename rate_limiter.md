# Rate Limiter

A rate limiter controls how many requests a client can make to an API within a specific time window. When a request comes in, the rate limiter checks if the client has exceeded their quota. If they're under the limit, the request proceeds. If they've hit the cap, the request gets rejected. This protects APIs from abuse and ensures fair resource allocation across clients.

---

## Requirements

You walk into the interview and get greeted with this prompt:

> "You're building an in-memory rate limiter for an API gateway. Each endpoint can have its own limit with a specific algorithm."

### Clarifying Questions

**You:** "The config includes an `algorithm` field. Are there different parameter shapes for different algorithms?"
**Interviewer:** "Yes. The `algoConfig` object always exists, but the fields inside vary by algorithm."

Good. Config is heterogeneous. The factory layer will need to extract the right parameters per algorithm.

**You:** "What information does each incoming request carry?"
**Interviewer:** "Client ID and endpoint. Client ID is just a unique string per caller."

**You:** "What should we return when checking a request? Just allowed/denied?"
**Interviewer:** "Return three things: whether it's allowed, how many requests remain in their quota, and if denied, when they can retry."

Now we know the return type is a structured object, not a boolean.

**You:** "What happens if a request comes in for an endpoint with no configuration?"
**Interviewer:** "Fall back to a default configuration. Don't reject requests just because config is missing."

**You:** "Should this handle concurrent requests from multiple threads?"
**Interviewer:** "Don't worry about it to start. We'll get to it if we have time."

**You:** "Single-process in-memory, or distributed across servers?"
**Interviewer:** "Single process, in-memory. Keep it simple."

**You:** "Is configuration loaded once at startup, or can it change at runtime?"
**Interviewer:** "Loaded once at startup. No hot-reloading."

### Final Requirements

| # | Requirement |
|---|-------------|
| 1 | Configuration is provided at startup (loaded once) |
| 2 | System receives requests with `(clientId: string, endpoint: string)` |
| 3 | Each endpoint has a config specifying algorithm and algorithm-specific parameters |
| 4 | System enforces rate limits by checking clientId against the endpoint's configuration |
| 5 | Return structured result: `(allowed: boolean, remaining: int, retryAfterMs: long \| null)` |
| 6 | If endpoint has no configuration, use a default limit |

**Out of scope:** distributed rate limiting, dynamic config updates, metrics/monitoring, UI layer.

---

## Core Entities and Relationships

The central challenge here isn't data modeling — it's algorithm pluggability. Each endpoint can use a different algorithm, each algorithm has different state, and we need to swap them without changing the rest of the system. That points immediately to a **Strategy pattern**.

**RateLimiter** — The orchestrator and public entry point. Receives `(clientId, endpoint)`, looks up the endpoint config, finds or lazily creates the right `RateLimitingStrategy` instance for that client, and delegates to it.

**RateLimitingStrategy (interface)** — The strategy abstraction. Naming it `RateLimitingStrategy` rather than just `Limiter` makes the pattern explicit to anyone reading the design. Each algorithm implements this. Owns its own per-client state and exposes a single `check()` method.

**TokenBucketLimiter** — One concrete strategy. Maintains a token count per client, refills over time, allows bursts up to capacity.

**LimiterFactory** — Reads an `EndpointConfig` and constructs the right `RateLimitingStrategy` implementation. The only place that knows about concrete algorithm classes.

**ClientEndpointKey** — A composite key that uniquely identifies a `(clientId, endpoint)` pair. Used as the key in the flat strategy map. Cleaner than a nested `Map<endpoint, Map<clientId, Strategy>>` and reduces lookup indirection.

**EndpointConfig** — A value object holding the algorithm name and its raw parameters. Parsed from the external config JSON.

**RateLimitResult** — The structured return value: allowed, remaining requests, and retry delay.

### Entity Responsibilities

| Entity | Responsibility |
|--------|----------------|
| `RateLimiter` | Entry point. Routes each request to the right `RateLimitingStrategy` instance. Manages lazy creation via `ClientEndpointKey`. |
| `RateLimitingStrategy` | Interface. Owns per-client algorithm state. Enforces the limit and computes remaining/retry info. |
| `TokenBucketLimiter` | Concrete strategy. Token bucket algorithm: allows bursts, refills at fixed rate. |
| `LimiterFactory` | Reads config, instantiates the right strategy subclass. Single place where algorithm names map to classes. |
| `ClientEndpointKey` | Composite map key. Bundles `clientId` + `endpoint` into one object with proper `equals`/`hashCode`. |
| `EndpointConfig` | Value object. Holds algorithm name and raw `algoConfig` parameters. |
| `RateLimitResult` | Value object. Carries `allowed`, `remaining`, `retryAfterMs`. |

### Relationships

```
RateLimiter
  ├── endpointConfigs: Map<endpoint, EndpointConfig>
  ├── strategies:      Map<ClientEndpointKey, RateLimitingStrategy>
  └── factory:         LimiterFactory
                           │
                           │ creates
                           v
                    <<interface>>
                  RateLimitingStrategy
                  └── check() -> RateLimitResult
                        ^              ^
                        |              |
            TokenBucketLimiter    SlidingWindowLogLimiter (future)
```

Each `(endpoint, clientId)` pair gets its own `RateLimitingStrategy` instance, keyed by a `ClientEndpointKey`, lazily created on first request. The algorithm and parameters come from `EndpointConfig`, but the per-client state (tokens, timestamps) lives inside the strategy instance.

---

## Class Design

We design top-down: `RateLimiter` first, then `RateLimitingStrategy` interface, then the concrete implementation, then the factory and value objects.

### RateLimiter

`RateLimiter` is what the API gateway calls. It routes each request to the right per-client strategy.

**Deriving State**

| Requirement | What RateLimiter must track |
|-------------|------------------------------|
| "Each endpoint has a configuration" | Map from endpoint → EndpointConfig |
| "Enforce limits per clientId per endpoint" | Map from ClientEndpointKey → RateLimitingStrategy |
| "Fall back to default config" | A default EndpointConfig when no match found |

Using a flat `Map<ClientEndpointKey, RateLimitingStrategy>` instead of a nested `Map<endpoint, Map<clientId, Strategy>>` reduces lookup to a single map access and avoids managing the inner map lifecycle.

**Deriving Behavior**

| Need from requirements | Method on RateLimiter |
|------------------------|-----------------------|
| "Check clientId against endpoint's config" | `check(clientId, endpoint) -> RateLimitResult` |

```
class RateLimiter:
    - endpointConfigs: Map<String, EndpointConfig>
    - strategies:      Map<ClientEndpointKey, RateLimitingStrategy>
    - factory:         LimiterFactory
    - defaultConfig:   EndpointConfig

    + RateLimiter(configs: List<EndpointConfig>, defaultConfig: EndpointConfig)
    + check(clientId, endpoint) -> RateLimitResult
```

Strategy instances are created **lazily** — only when a client first hits an endpoint. Pre-creating strategies for all possible clients is impossible (we don't know them upfront) and wasteful.

---

### RateLimitingStrategy (interface)

Every algorithm implements this. Each instance represents one client's state for one endpoint.

The name `RateLimitingStrategy` signals the pattern immediately — anyone reading the design sees "this is the strategy interface, concrete algorithms plug in here." A generic name like `Limiter` conveys the same structure but obscures the intent.

**Deriving Behavior**

| Need | Method on RateLimitingStrategy |
|------|-------------------------------|
| "Check if request is allowed, return remaining and retry info" | `check() -> RateLimitResult` |

```
interface RateLimitingStrategy:
    + check() -> RateLimitResult
```

No `clientId` parameter on `check()` — each instance already IS the per-client state. `RateLimiter` routes to the right instance before calling `check()`.

---

### TokenBucketLimiter implements RateLimitingStrategy

Token Bucket maintains a floating token count. Tokens accumulate up to `capacity` at `refillRatePerSecond`. Each request costs one token. Allows bursts (drain the full bucket) while capping sustained throughput.

**Deriving State**

| Requirement | What TokenBucketLimiter must track |
|-------------|-------------------------------------|
| "Capacity" from algoConfig | Current token count and max capacity |
| "RefillRatePerSecond" from algoConfig | Rate at which tokens are added |
| Compute tokens added since last check | Timestamp of last refill |

```
class TokenBucketLimiter implements RateLimitingStrategy:
    - tokens:              double
    - capacity:            int
    - refillRatePerSecond: double
    - lastRefillTime:      long    // epoch ms

    + TokenBucketLimiter(capacity, refillRatePerSecond)
    + check() -> RateLimitResult
```

`tokens` is a `double` (not int) because partial tokens accumulate between refills. A rate of 10/sec means 1 token per 100ms. If `check()` is called 50ms after last refill, 0.5 tokens have accrued. We track that fraction and it contributes to the next call.

---

### LimiterFactory

Factory reads an `EndpointConfig` and constructs the right `RateLimitingStrategy`. The only class that knows about concrete algorithm implementations.

**Deriving Behavior**

| Need | Method on LimiterFactory |
|------|--------------------------|
| "Construct a strategy from config" | `create(config) -> RateLimitingStrategy` |

```
class LimiterFactory:
    + create(config: EndpointConfig) -> RateLimitingStrategy
```

---

### Value Objects

```
class ClientEndpointKey:
    - clientId: String
    - endpoint: String

    + ClientEndpointKey(clientId, endpoint)
    + equals(other) -> boolean    // must compare both fields
    + hashCode()    -> int        // must hash both fields

class EndpointConfig:
    - endpoint:   String
    - algorithm:  String
    - algoConfig: Map<String, Object>

    + EndpointConfig(endpoint, algorithm, algoConfig)
    + getEndpoint()   -> String
    + getAlgorithm()  -> String
    + getAlgoConfig() -> Map<String, Object>

class RateLimitResult:
    - allowed:      boolean
    - remaining:    int
    - retryAfterMs: long?    // null if allowed

    + RateLimitResult(allowed, remaining, retryAfterMs)
    + isAllowed()       -> boolean
    + getRemaining()    -> int
    + getRetryAfterMs() -> long?
```

`ClientEndpointKey` **must** override `equals()` and `hashCode()`. Without both, two `ClientEndpointKey("alice", "/search")` objects would be treated as different map keys even though they represent the same logical pair. In Java, for example, this is done by annotating with `@EqualsAndHashCode` (Lombok) or implementing both methods manually.

---

### Final Class Design

```
+-------------------------------+          +-------------------------------+
|         RateLimiter           |          |         LimiterFactory        |
+-------------------------------+          +-------------------------------+
| - endpointConfigs: Map        |--------->| + create(config)              |
| - strategies:                 |          |     -> RateLimitingStrategy   |
|     Map<ClientEndpointKey,    |          +-------------------------------+
|     RateLimitingStrategy>     |                          |
| - factory: LimiterFactory     |                          | creates
| - defaultConfig: EndpointConfig|                         v
+-------------------------------+              +-------------------------+
| + check(clientId, endpoint)   |              |    <<interface>>        |
+-------------------------------+              |  RateLimitingStrategy   |
                                               +-------------------------+
                                               | + check()               |
                                               |     -> RateLimitResult  |
                                               +-------------------------+
                                                      ^           ^
                                                      |           |
                                   +------------------+           +-------------------+
                                   |                                                  |
                      +------------+-----------+          +------------------------+  |
                      |   TokenBucketLimiter   |          | SlidingWindowLogLimiter|  |
                      +------------------------+          +------------------------+  |
                      | - tokens: double       |          | - timestamps: Queue    |  |
                      | - capacity: int        |          | - limit: int           |  |
                      | - refillRate: double   |          | - windowMs: long       |  |
                      | - lastRefillTime: long |          +------------------------+  |
                      +------------------------+          | + check()              |  |
                      | + check()              |          +------------------------+  |
                      +------------------------+

+---------------------------+     +---------------------------+     +---------------------------+
|    ClientEndpointKey      |     |      EndpointConfig       |     |      RateLimitResult      |
+---------------------------+     +---------------------------+     +---------------------------+
| - clientId: String        |     | - endpoint: String        |     | - allowed: boolean        |
| - endpoint: String        |     | - algorithm: String       |     | - remaining: int          |
+---------------------------+     | - algoConfig: Map         |     | - retryAfterMs: long?     |
| + equals() / hashCode()   |     +---------------------------+     +---------------------------+
+---------------------------+
```

---

## Implementation

The two most interesting methods are `RateLimiter.check()` (routing and lazy creation) and `TokenBucketLimiter.check()` (the algorithm itself). We'll also walk through `LimiterFactory.create()`.

### RateLimiter.check()

**Core logic:**
1. Look up config for the endpoint (fall back to default if missing)
2. Build a `ClientEndpointKey` for this `(clientId, endpoint)` pair
3. Find or lazily create the strategy for that key
4. Delegate to `strategy.check()`

**Edge cases:**
- Endpoint not in config → use defaultConfig
- Client hitting this endpoint for the first time → create strategy, store it

```
check(clientId, endpoint)
    config   = endpointConfigs.getOrDefault(endpoint, defaultConfig)
    key      = ClientEndpointKey(clientId, endpoint)

    if !strategies.containsKey(key)
        strategies[key] = factory.create(config)

    strategy = strategies[key]
    return strategy.check()
```

All algorithm logic is in `RateLimitingStrategy`. `RateLimiter.check()` has no algorithm knowledge — it only handles routing and lazy initialization. The flat map with a composite key keeps this simple: one lookup, no inner map management.

---

### TokenBucketLimiter.check()

This is the heart of the Token Bucket algorithm.

**Core logic:**
1. Compute elapsed time since last refill
2. Add tokens proportional to elapsed time, cap at capacity
3. If tokens ≥ 1: consume one token, return allowed
4. If tokens < 1: compute when next token arrives, return denied

**Edge cases:**
- First call: `tokens` starts at `capacity` (bucket starts full)
- Long gap between requests: tokens accumulate but never exceed capacity
- Rapid requests: tokens drain to 0 and requests are denied

```
check()
    now     = currentTimeMs()
    elapsed = (now - lastRefillTime) / 1000.0      // convert ms to seconds
    tokens  = min(tokens + elapsed * refillRatePerSecond, capacity)
    lastRefillTime = now

    if tokens >= 1
        tokens -= 1
        return RateLimitResult(true, floor(tokens), null)
    else
        msUntilToken = ((1 - tokens) / refillRatePerSecond) * 1000
        return RateLimitResult(false, 0, ceil(msUntilToken))
```

`retryAfterMs` tells the client exactly how long to wait before a token is available. `(1 - tokens)` is the fractional deficit — if we have 0.3 tokens and need 1.0, we need 0.7 more tokens at `refillRatePerSecond`.

---

### LimiterFactory.create()

The only place that maps algorithm names to classes.

**Core logic:**
- Switch on algorithm name
- Extract the right parameters from `algoConfig`
- Construct and return the concrete strategy

**Edge cases:**
- Unknown algorithm name → throw an error at startup (fail fast, not at request time)

```
create(config)
    algo   = config.getAlgorithm()
    params = config.getAlgoConfig()

    switch algo
        case "TokenBucket":
            capacity   = params["capacity"]
            refillRate = params["refillRatePerSecond"]
            return TokenBucketLimiter(capacity, refillRate)

        case "SlidingWindowLog":
            limit    = params["limit"]
            windowMs = params["windowMs"]
            return SlidingWindowLogLimiter(limit, windowMs)

        default:
            throw Error("Unknown algorithm: " + algo)
```

Failing fast on unknown algorithms at config load time is deliberate. Discovering a misconfigured algorithm only when a request arrives would silently fall through to a default or crash mid-traffic.

---

## Design Patterns

### Strategy Pattern
`RateLimitingStrategy` is the strategy interface. `TokenBucketLimiter` and `SlidingWindowLogLimiter` are concrete strategies. `RateLimiter` holds one strategy per `ClientEndpointKey` without knowing which algorithm is behind it.

Adding a new algorithm means:
1. Implement `RateLimitingStrategy`
2. Register it in `LimiterFactory`
3. Done — `RateLimiter` and all other classes are untouched

### Factory Pattern
`LimiterFactory` centralizes the mapping from config string → concrete class. Without it, that `switch` statement would live in `RateLimiter.check()`, mixing routing logic with construction logic.

### Information Expert
- `TokenBucketLimiter` owns `tokens` and `lastRefillTime` → it computes remaining and retryAfterMs
- `RateLimiter` owns `endpointConfigs` → it decides which config applies
- `LimiterFactory` knows the algorithm registry → it decides which class to instantiate

---

## Verification

Config: `/search` → TokenBucket(capacity=3, refillRatePerSecond=1). DefaultConfig → TokenBucket(capacity=10, refillRatePerSecond=1).

**Client "alice" makes 3 rapid requests to /search (t=0ms):**
```
check("alice", "/search")   // Request 1
  config   = endpointConfigs["/search"]
  key      = ClientEndpointKey("alice", "/search")
  strategy = NEW TokenBucketLimiter(3, 1)   // first time, create and store
  strategy.check():
    elapsed=0, tokens=min(3+0, 3)=3
    tokens >= 1 -> tokens=2
  Result: allowed=true, remaining=2, retryAfterMs=null

check("alice", "/search")   // Request 2
  key=same, strategy=existing
  strategy.check(): elapsed~0, tokens=2 -> tokens=1
  Result: allowed=true, remaining=1, retryAfterMs=null

check("alice", "/search")   // Request 3
  strategy.check(): elapsed~0, tokens=1 -> tokens=0
  Result: allowed=true, remaining=0, retryAfterMs=null
```

**4th request immediately (t~0ms):**
```
check("alice", "/search")   // Request 4
  strategy.check(): elapsed~0, tokens=0
  tokens < 1
  msUntilToken = ((1 - 0) / 1) * 1000 = 1000ms
  Result: allowed=false, remaining=0, retryAfterMs=1000
```

**Request from "bob" — different key, independent strategy:**
```
check("bob", "/search")
  key      = ClientEndpointKey("bob", "/search")   // different key
  strategy = NEW TokenBucketLimiter(3, 1)           // bob gets his own bucket
  strategy.check(): tokens=3 -> tokens=2
  Result: allowed=true, remaining=2, retryAfterMs=null
```

**Request to unknown endpoint /unknown:**
```
check("alice", "/unknown")
  config = defaultConfig (no entry in endpointConfigs)
  key    = ClientEndpointKey("alice", "/unknown")
  strategy = NEW TokenBucketLimiter(10, 1)
  Result: allowed=true, remaining=9, retryAfterMs=null
```

The trace confirms: per-client isolation via `ClientEndpointKey`, token depletion, correct deny with retry hint, default config fallback.

---

## Extensibility

### 1. "How would you add a new algorithm — say, Fixed Window Counter?"

> "I'd implement `RateLimitingStrategy` with a `FixedWindowLimiter` class. It tracks a request count and a window start time. When the window expires, it resets the count. Then I'd add one case to `LimiterFactory.create()`. Nothing else changes — `RateLimiter`, `ClientEndpointKey`, `EndpointConfig`, and `RateLimitResult` are all untouched. This is the Strategy pattern working as intended: adding an algorithm is purely additive."

```
class FixedWindowLimiter implements RateLimitingStrategy:
    - count:       int
    - limit:       int
    - windowMs:    long
    - windowStart: long

    + check() -> RateLimitResult
        now = currentTimeMs()
        if now - windowStart >= windowMs
            count = 0
            windowStart = now
        if count < limit
            count++
            return RateLimitResult(true, limit - count, null)
        else
            retryAfterMs = windowStart + windowMs - now
            return RateLimitResult(false, 0, retryAfterMs)

// In LimiterFactory:
case "FixedWindow":
    return FixedWindowLimiter(params["limit"], params["windowMs"])
```

---

### 2. "How would you support per-client custom limits?"

Right now all clients hitting the same endpoint share the same config. Some APIs give premium clients higher limits.

> "I'd change the config lookup key. Currently `endpointConfigs` maps `endpoint → config`. I'd change it to a `(endpoint, clientId) → config` map, with a fallback to the endpoint-level config, then the global default. The `RateLimitingStrategy` interface and all algorithm implementations are completely unchanged. The only change is in `RateLimiter.check()` — try the client-specific config first, then fall back."

```
check(clientId, endpoint)
    specificKey = endpoint + ":" + clientId
    config = endpointConfigs.getOrDefault(specificKey,
               endpointConfigs.getOrDefault(endpoint, defaultConfig))
    // rest of the method unchanged
```

---

### 3. "How would you make this thread-safe?"

Right now the system is single-threaded. In a real API gateway, many threads call `check()` concurrently on the same `RateLimiter`.

Two failure modes to protect against:
- **Lazy creation race** — two threads both see a missing strategy entry for the same `ClientEndpointKey` and both try to create one. One wins, the other's state is lost.
- **Algorithm state race** — inside `TokenBucketLimiter.check()`, reading `tokens` and writing it back is not atomic. Two threads can both see 1 token, both consume it, and both return `allowed=true` when only one should.

> "I'd use two levels of synchronization. For lazy creation, `ConcurrentHashMap.computeIfAbsent()` on the flat `ClientEndpointKey` map guarantees exactly one strategy is created per key even under concurrent access — and this is now a single-level lookup, no nested map to worry about. For algorithm state inside each strategy, I'd add a `synchronized` block (or `ReentrantLock`) around the body of `check()`. Each strategy instance has its own lock, so two different clients never contend — only concurrent calls from the same client to the same endpoint need to serialize."

```
// In RateLimiter — safe lazy creation (flat map makes this clean)
check(clientId, endpoint)
    config   = endpointConfigs.getOrDefault(endpoint, defaultConfig)
    key      = ClientEndpointKey(clientId, endpoint)
    strategy = strategies.computeIfAbsent(key, _ -> factory.create(config))
    return strategy.check()

// In TokenBucketLimiter — safe state mutation
check()
    synchronized(this)
        // ... token refill and consume logic unchanged
```

The flat `ClientEndpointKey` map pays off here too. With a nested map, `computeIfAbsent` on the outer map doesn't protect the inner map creation. With a flat map and a composite key, a single `computeIfAbsent` call atomically handles the whole lookup and creation.
