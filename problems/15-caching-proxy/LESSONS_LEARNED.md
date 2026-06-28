# Lessons Learned — Caching Proxy

---

## Mistake 1: Proxy Does Not Implement Same Interface

**What happens:** `CachingUserService` has its own method names or wraps the real service without a shared interface.

**Why it's wrong:** Client code must change to use the cache. The defining feature of Proxy is transparency — client can't tell if it's talking to proxy or real service.

**Fix:** `UserServiceProxy implements UserService` — same interface, zero client changes.

**Gap dimension:** Design Patterns (Dimension 3)

---

## Mistake 2: HashMap Instead of ConcurrentHashMap

**What happens:** `Map<String, CacheEntry> cache = new HashMap<>()`.

**Why it's wrong:** `HashMap` is not thread-safe. Concurrent `put` operations can corrupt the internal hash table structure. `get` concurrent with structural modifications can loop infinitely (Java 7 bug, fixed in Java 8 but still not safe).

**Fix:** `ConcurrentHashMap<String, CacheEntry>` — provides segment-level locking; safe concurrent reads and writes.

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 3: Cache Never Expires (No TTL)

**What happens:** `cache.put(key, value)` — no expiry. Stale data returned forever.

**Why it's wrong:** User profile changes in the DB but proxy keeps returning the old version. For anything privacy-sensitive (deleted accounts, changed permissions) this is a correctness issue.

**Fix:** `CacheEntry` with `expiresAt = now + ttlMs`:
```java
CacheEntry entry = cache.get(userId);
if (entry != null && !entry.isExpired()) return entry.value; // valid hit
// otherwise: miss — call real service
```

**Gap dimension:** Requirements / Entity Modeling (Dimensions 1, 2)

---

## Mistake 4: Thundering Herd — Duplicate DB Calls on Concurrent Miss

**What happens:** 100 threads miss the cache for key "42". All call the real service. 100 DB calls fire.

**Why it's wrong:** The entire point of caching is to reduce load on the backend. Under traffic spikes (exactly when cache matters most), the thundering herd can overwhelm the DB.

**Fix:** Cache the `Future`, not the value:
```java
FutureTask<String> ft = new FutureTask<>(() -> delegate.getUserData(userId));
Future<String> existing = cache.putIfAbsent(userId, ft); // atomic
if (existing == null) { ft.run(); } // winner: do the work
else { return existing.get(); }     // others: wait on same Future
```

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 5: Missing `volatile` on Virtual Proxy's Lazy-Init Field

**What happens:** `private RealImage realImage;` — no `volatile`. Double-checked locking used.

**Why it's wrong:** Without `volatile`, the JVM can reorder object construction steps. Thread A writes the pointer to `realImage` before the object is fully initialized. Thread B reads a non-null but incompletely constructed object. Corrupted state.

**Fix:** `private volatile RealImage realImage;` — `volatile` establishes a happens-before relationship: writes to the object's fields before the volatile write are visible to any thread that reads the volatile field.

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 6: Caching Failure Results

**What happens:** Real service throws an exception. Proxy catches it, stores a sentinel in cache, and returns null (or the sentinel) on all subsequent calls.

**Why it's wrong:** A transient network error poisons the cache. All calls for that key fail permanently until TTL expires.

**Fix:** On exception, `cache.remove(key)` and rethrow. Don't cache failures.
```java
try {
    String data = delegate.getUserData(userId);
    cache.put(userId, new CacheEntry(data, ttlMs));
    return data;
} catch (Exception e) {
    cache.remove(userId); // don't poison
    throw e;
}
```

**Gap dimension:** Requirements (Dimension 1)

---

## Self-Assessment Checklist

- [ ] Proxy implements same interface as real service?
- [ ] `ConcurrentHashMap` used (not `HashMap`)?
- [ ] `CacheEntry` has `expiresAt` field and `isExpired()` check?
- [ ] Thundering herd addressed (at minimum recognized and named)?
- [ ] Failed real-service calls don't populate cache?
- [ ] `volatile` on Virtual Proxy's lazy-init field?
- [ ] Proxy is transparent to client (zero client code changes)?

---

## Remediation Targets

| Dimension | If scored 0–1 | Go to |
|-----------|--------------|-------|
| Design Patterns | Wrong interface or no shared interface | GAP_REMEDIATION.md#gap-3 — Proxy trigger: "control access, same interface" |
| Concurrency | HashMap, thundering herd, missing volatile | GAP_REMEDIATION.md#gap-6 — shared state audit |
| Requirements | No TTL, cache poisons on failure | GAP_REMEDIATION.md#gap-1 — requirements drill |
