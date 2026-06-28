# Caching Proxy

**Difficulty:** Intermediate
**Category:** Proxy Pattern
**Time Box:** 45 min (discussion) / 90 min (machine coding)
**Key Patterns:** Proxy (Virtual + Caching), Decorator (alternative framing)
**Asked at:** Google, Amazon, Adobe, LinkedIn, Stripe

---

## Problem Statement

Design a caching proxy for a slow external service (e.g., a user profile service that hits a database on every call). The proxy sits between the client and the real service, implementing the same interface. It caches responses so that repeated calls for the same input skip the expensive operation and return the cached result. The proxy should be transparent to the client.

This problem also covers Virtual Proxy (lazy initialization of an expensive object) as a secondary variant.

---

## Actors

| Actor | Description |
|-------|-------------|
| Client | Calls the service interface; unaware of whether it gets proxy or real service |
| Proxy | Intercepts calls; checks cache; delegates to real service on cache miss |
| Real Service | Expensive operation (DB query, API call, disk I/O) |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: Proxy implements the same interface as the real service
- FR2: On cache hit, return cached result immediately (no call to real service)
- FR3: On cache miss, call real service, cache the result, return it
- FR4: Cache entries expire after a configurable TTL (Time-To-Live)
- FR5: Thread-safe — concurrent calls for the same key should not trigger duplicate real-service calls
- FR6: Client code requires zero changes when switching from real service to proxy

**Secondary (implement if time allows):**
- FR7: Cache eviction — LRU or max-size cap
- FR8: Cache invalidation — explicit `invalidate(key)` method
- FR9: Metrics — cache hit rate tracking
- FR10: Virtual Proxy variant — lazy initialization of the real service on first use

---

## Non-Functional Requirements

- **Transparency:** Proxy and real service share the exact same interface
- **Thread safety:** Multiple threads calling `getData(key)` concurrently on the same key must not cause duplicate real-service calls (thundering herd problem)
- **Correctness:** Expired entries must not be returned

---

## Constraints and Assumptions

- Real service is slow (simulate with `Thread.sleep` or similar)
- Cache stored in-memory (`ConcurrentHashMap`)
- TTL configurable at proxy construction
- Keys and values are strings for MVP (generics are a bonus)
- Out of scope: distributed cache (Redis), cache serialization, write-through cache

---

## Good Clarifying Questions to Ask

1. What's the interface — a single `getData(id)`, or multiple methods?
2. Should the cache expire entries by TTL, or by size (LRU)?
3. Thread safety required — how many concurrent callers?
4. What if the real service call fails — cache the failure or throw?
5. Should failed lookups be retried or cached as negative results?
6. Is explicit cache invalidation needed?

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **UserService** (Subject interface) — `getUserData(String userId): String`
- **RealUserService** (Real Subject) — slow implementation; hits DB; never changes
- **UserServiceProxy** (Proxy) — implements `UserService`; wraps `RealUserService`; manages `ConcurrentHashMap<String, CacheEntry>`
- **CacheEntry** — holds `value: String` + `expiresAt: long`; `isExpired(): boolean`
- **ThunderingHerd fix** — `ConcurrentHashMap<String, Future<String>>` to prevent duplicate parallel calls to real service

</details>

---

## Proxy vs Decorator — Know the Difference

Both wrap an object and implement the same interface. The distinction:

| | Proxy | Decorator |
|---|---|---|
| Purpose | Control ACCESS to the object | ADD BEHAVIOUR to the object |
| Client awareness | Client thinks it's talking to the real thing | Client knows it's getting enhanced |
| Object lifecycle | Proxy often manages the real object's lifecycle | Decorator receives the object from outside |
| Examples | Cache proxy, lazy init, auth gate | BufferedReader wrapping FileReader, rate-limiting channel |
