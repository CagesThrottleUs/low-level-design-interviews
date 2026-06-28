# Solution Guide — Caching Proxy

**Read after your attempt.**

---

## Entity Map

| Class/Interface | Type | Responsibility |
|----------------|------|---------------|
| `UserService` | Interface | Contract: `String getUserData(String userId)` |
| `RealUserService` | Concrete (Real Subject) | Slow DB call; never modified |
| `UserServiceProxy` | Proxy | `ConcurrentHashMap<String, CacheEntry>` + TTL; delegates to real on miss |
| `CacheEntry` | Value object | `value: String`, `expiresAt: long`, `isExpired()` |

---

## Basic Caching Proxy

```java
// Subject interface — proxy and real service share this
public interface UserService {
    String getUserData(String userId);
}

// Real service — expensive, never modified
public class RealUserService implements UserService {
    @Override
    public String getUserData(String userId) {
        System.out.println("Hitting DB for user: " + userId);
        try { Thread.sleep(2000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        return "UserData:" + userId;
    }
}

// TTL-bearing cache entry
public class CacheEntry {
    final String value;
    final long expiresAt;

    CacheEntry(String value, long ttlMs) {
        this.value = value;
        this.expiresAt = System.currentTimeMillis() + ttlMs;
    }

    boolean isExpired() { return System.currentTimeMillis() > expiresAt; }
}

// Proxy — same interface, adds caching transparently
public class UserServiceProxy implements UserService {
    private final UserService delegate;
    private final ConcurrentHashMap<String, CacheEntry> cache = new ConcurrentHashMap<>();
    private final long ttlMs;

    public UserServiceProxy(UserService delegate, long ttlMs) {
        this.delegate = delegate;
        this.ttlMs    = ttlMs;
    }

    @Override
    public String getUserData(String userId) {
        CacheEntry entry = cache.get(userId);
        if (entry != null && !entry.isExpired()) {
            System.out.println("Cache hit for: " + userId);
            return entry.value;
        }
        // Cache miss or expired — call real service
        System.out.println("Cache miss for: " + userId);
        String data = delegate.getUserData(userId);
        cache.put(userId, new CacheEntry(data, ttlMs));
        return data;
    }

    public void invalidate(String userId) {
        cache.remove(userId);
    }
}
```

**Client code — zero changes needed:**
```java
UserService service = new UserServiceProxy(new RealUserService(), 60_000L);
// Client calls same interface regardless of proxy or real
System.out.println(service.getUserData("42")); // cache miss → 2s
System.out.println(service.getUserData("42")); // cache hit → instant
```

---

## Thread Safety: Thundering Herd Problem

**Problem:** 100 threads simultaneously request user "42" which isn't in cache. All get a cache miss, all call the DB, 100 DB calls fired for one key.

**Level 1 fix: synchronized block (simple, correct, but serializes all misses)**
```java
public synchronized String getUserData(String userId) {
    CacheEntry entry = cache.get(userId);
    if (entry != null && !entry.isExpired()) return entry.value;
    String data = delegate.getUserData(userId);
    cache.put(userId, new CacheEntry(data, ttlMs));
    return data;
}
// Safe but: one thread calling DB blocks all other threads for all keys
```

**Level 2 fix: Future-based (lock-free, prevents thundering herd)**
```java
public class UserServiceProxy implements UserService {
    // Cache the Future, not the value — concurrent misses wait on ONE computation
    private final ConcurrentHashMap<String, Future<String>> cache = new ConcurrentHashMap<>();
    private final ExecutorService executor = Executors.newCachedThreadPool();
    private final UserService delegate;

    @Override
    public String getUserData(String userId) {
        Future<String> f = cache.get(userId);
        if (f == null) {
            FutureTask<String> ft = new FutureTask<>(() -> delegate.getUserData(userId));
            f = cache.putIfAbsent(userId, ft); // atomic: only first caller gets null back
            if (f == null) {
                f = ft;
                ft.run(); // this thread does the work
            }
            // All other threads: f != null, they wait on the same Future
        }
        try {
            return f.get(); // blocks until result ready
        } catch (Exception e) {
            cache.remove(userId); // don't poison cache on failure
            throw new RuntimeException("Failed to get user data", e);
        }
    }
}
```

**Why `putIfAbsent` is the key:** Atomically inserts only if key absent. Only ONE thread gets `null` back (the winner). All others get the existing `Future` and wait on it. One DB call total.

---

## Virtual Proxy Variant

```java
// Expensive object — should not be created until first use
public class RealHeavyImage implements Image {
    private final byte[] pixels;

    public RealHeavyImage(String filename) {
        System.out.println("Loading " + filename + " from disk...");
        this.pixels = loadFromDisk(filename); // slow
    }

    @Override public void display() { /* render pixels */ }
}

public class ProxyImage implements Image {
    private volatile RealHeavyImage realImage; // volatile for DCL correctness
    private final String filename;

    public ProxyImage(String filename) {
        this.filename = filename;
        // Real image NOT loaded here — that's the whole point
    }

    @Override
    public void display() {
        if (realImage == null) {
            synchronized (this) {
                if (realImage == null) {           // double-checked locking
                    realImage = new RealHeavyImage(filename);
                }
            }
        }
        realImage.display();
    }
}
```

`volatile` is mandatory for correctness: without it, CPU reordering can expose a partially-constructed `RealHeavyImage` to another thread between the pointer being written and the object being fully initialized.

---

## Extension Points

| "Now add..." | Change |
|-------------|--------|
| LRU eviction | Swap `ConcurrentHashMap` for `LinkedHashMap` + `removeEldestEntry` override; or use Caffeine |
| Max size cap | Track `size` atomically; evict on overflow |
| Cache invalidation | `cache.remove(key)` — already shown |
| Protection proxy (auth) | Check `currentUser.hasPermission()` before delegating |
| Metrics (hit rate) | `AtomicLong hits, misses` incremented per call |

---

## Proxy vs Decorator vs Adapter

| Pattern | Changes interface? | Adds behaviour? | Controls access? |
|---------|-------------------|-----------------|-----------------|
| Proxy | No | Sometimes | Yes |
| Decorator | No | Yes | No |
| Adapter | Yes | No | No |

---

## What Strong Candidates Do Differently

- `volatile` on lazy-init field in Virtual Proxy — explains JMM reason
- `putIfAbsent` + `FutureTask` for thundering herd — explains the N-simultaneous-miss scenario
- `cache.remove(key)` on real service failure — prevents cache poisoning
- Generic `<K, V>` type parameters — applicable to any service
- Explicitly state: "Proxy is transparent to client — same interface, zero client changes"

## What Average Candidates Miss

- `HashMap` instead of `ConcurrentHashMap` — race on concurrent writes
- TTL expiry not checked — stale data returned forever
- No thundering herd handling — 1000 DB calls for 1000 concurrent misses on same key
- Real service object created eagerly in Virtual Proxy constructor — defeats lazy loading
- Missing `volatile` on DCL field — silent data race
