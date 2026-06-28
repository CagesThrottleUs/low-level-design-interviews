# Mock Interview Script — Caching Proxy

**For the interviewer.**

---

## Opening

> "Design a caching proxy for a slow user profile service. The service hits a database on every call. Add a caching layer that returns cached results for repeated requests. The caller should not need to change their code to use the cache. Ask clarifying questions first."

| If candidate asks | Answer |
|------------------|--------|
| What interface? | `String getUserData(String userId)` — one method for MVP |
| Cache expiry? | TTL — configurable at proxy construction (e.g., 60 seconds) |
| Thread safety? | Yes — multiple threads call this concurrently |
| Eviction policy? | TTL expiry for MVP; LRU is secondary |
| Cache invalidation? | Yes — `invalidate(userId)` should be supported |
| Failure handling? | If real service throws, don't cache the failure — let exception propagate |

---

## Phase 2: Core Design — What to Watch For

**Key signal 1: Same interface**

Good: `UserServiceProxy implements UserService` — client calls the same method signature whether talking to proxy or real service.

Bad: `CachingUserService` with a different method name or signature — breaks transparency.

**Key signal 2: `ConcurrentHashMap` not `HashMap`**

Good: "Multiple threads can call `getUserData` simultaneously — I need `ConcurrentHashMap`."

**Key signal 3: TTL check**

Good: `CacheEntry` with `expiresAt = System.currentTimeMillis() + ttlMs`. Checked on every `get`.

Bad: Cache never expires — stale data returned forever.

---

## Extension Probe (~35 min)

> "What happens if 500 threads all request the same user ID simultaneously, and it's a cache miss for all of them? How many DB calls fire?"

Expected progression:
- Level 0: "All 500 hit the DB" (thundering herd — wrong)
- Level 1: Recognize the problem: "I should prevent duplicate calls"
- Level 2: Synchronized block — "Only one thread at a time, serialized" (correct but slow)
- Level 3: `FutureTask` + `putIfAbsent` — "One thread computes, 499 wait on same Future" (optimal)

> "How does Proxy differ from Decorator? Both wrap an object with the same interface."

Expected: Proxy controls ACCESS (lazy init, caching, auth gate). Decorator adds BEHAVIOUR (buffering, logging, rate limiting). Proxy often manages the real object's lifecycle; Decorator receives the object from outside.

---

## Concurrency Probe (if not raised)

> "Walk me through what happens when Thread A and Thread B both miss the cache for key '42' simultaneously."

With basic `ConcurrentHashMap`:
- Both get `null` on `cache.get("42")`
- Both call `delegate.getUserData("42")`
- Both call `cache.put("42", ...)`
- 2 DB calls instead of 1 — thundering herd for 2 threads; 1000 calls for 1000 threads

Fix: `putIfAbsent(key, futureTask)` — atomic check-and-insert. Only one thread wins; others get the existing Future and wait.

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | TTL expiry, invalidation, transparency, thread safety | |
| Entity Modeling | `CacheEntry` with `expiresAt`; Proxy implements same interface | |
| Design Patterns | Proxy (same interface, controls access) vs Decorator explained | |
| SOLID | OCP: new caching behavior = new proxy class; real service untouched | |
| Extensibility | LRU eviction; metrics; protection proxy — all addable without changing real service | |
| Concurrency | `ConcurrentHashMap`, thundering herd recognized + fix described | |
| Communication | Explained WHY Proxy (transparency), WHY `volatile` on lazy init | |
| **TOTAL** | | **/21** |
