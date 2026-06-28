# Proxy Pattern

**Category:** Structural
**LLD interview frequency:** Tier 2
**Problem:** 15 — Caching Proxy

---

## The Problem It Solves

You want to add behaviour around an object (caching, access control, lazy loading) without modifying the object's class, and without the client knowing or caring that it's talking to a wrapper.

```java
// Without Proxy — client forced to handle caching logic itself
class ReportController {
    private UserService userService;
    private Map<String, String> cache = new HashMap<>();

    String getUser(String id) {
        if (cache.containsKey(id)) return cache.get(id); // caching in controller!
        String user = userService.getUserData(id);
        cache.put(id, user);
        return user;
    }
    // Caching logic repeated in every controller that uses UserService
    // Testing ReportController requires handling cache state too
}
```

---

## Intent

Provide a substitute or placeholder for another object. The proxy controls access to the original object, allowing actions before or after requests reach it.

---

## Trigger Signals

Apply Proxy when you see:
- "Add caching/logging/auth without changing the existing class"
- "Delay expensive object creation until first use"
- "Control access — not everyone can call this"
- Same interface must be maintained (client code must not change)

---

## Three Types

### Virtual Proxy — Lazy Initialization

Create the expensive object only when first needed.

```java
public class ProxyImage implements Image {
    private volatile RealImage realImage; // volatile for DCL correctness
    private final String filename;

    public ProxyImage(String filename) {
        this.filename = filename;
        // RealImage NOT created here — that's the point
    }

    @Override
    public void display() {
        if (realImage == null) {
            synchronized (this) {
                if (realImage == null) {           // double-checked locking
                    realImage = new RealImage(filename); // expensive — only on first call
                }
            }
        }
        realImage.display();
    }
}
```

`volatile` is mandatory: without it, CPU reordering can expose a partially-constructed object.

---

### Caching Proxy — Skip Expensive Calls

Cache results so repeated calls skip the real operation.

```java
public class UserServiceProxy implements UserService {
    private final UserService delegate;
    private final ConcurrentHashMap<String, CacheEntry> cache = new ConcurrentHashMap<>();
    private final long ttlMs;

    @Override
    public String getUserData(String userId) {
        CacheEntry entry = cache.get(userId);
        if (entry != null && !entry.isExpired()) {
            return entry.value; // cache hit
        }
        String data = delegate.getUserData(userId); // cache miss — call real service
        cache.put(userId, new CacheEntry(data, ttlMs));
        return data;
    }

    public void invalidate(String userId) { cache.remove(userId); }
}
```

---

### Protection Proxy — Access Control

Check permissions before delegating.

```java
public class SecureFileSystemProxy implements FileSystem {
    private final FileSystem delegate;
    private final User currentUser;

    @Override
    public void writeFile(String path, String content) {
        if (!currentUser.hasPermission(path, Permission.WRITE)) {
            throw new AccessDeniedException("User " + currentUser + " cannot write to " + path);
        }
        delegate.writeFile(path, content);
    }
}
```

---

## Thundering Herd Problem

When many threads simultaneously miss the cache for the same key, they all call the real service:

```java
// Problem: 100 threads miss on key "42" → 100 DB calls
CacheEntry entry = cache.get("42"); // all 100 get null
String data = delegate.getUserData("42"); // all 100 call DB

// Fix: cache the Future, not the value
FutureTask<String> ft = new FutureTask<>(() -> delegate.getUserData(userId));
Future<String> existing = cache.putIfAbsent(userId, ft); // atomic: only 1 wins
if (existing == null) { ft.run(); } // winner computes
return (existing != null ? existing : ft).get(); // others wait on same Future
```

---

## Proxy vs Decorator vs Adapter

All three wrap an object. The distinction is intent:

| | Proxy | Decorator | Adapter |
| --- | --- | --- | --- |
| Changes interface? | No | No | Yes |
| Purpose | Control ACCESS | Add BEHAVIOUR | Convert interface |
| Object lifecycle | Proxy often manages real object | Decorator receives object from caller | Adapter wraps incompatible object |
| Example | Cache proxy, auth gate | `BufferedInputStream(FileInputStream)` | `InputStreamReader(InputStream)` |

---

## Underlying Principle

**OCP:** Real service is never modified. New proxy behaviours = new proxy classes.
**DIP:** Client depends on `UserService` interface. Proxy injects transparently.
**SRP:** Proxy handles one cross-cutting concern (caching OR auth OR logging — not all three).

---

## Real-World Usage

- **Spring `@Transactional`** — Spring wraps your bean with a transaction proxy at runtime
- **Spring `@Cacheable`** — Spring wraps your bean with a cache proxy
- **JDK dynamic proxy** — `Proxy.newProxyInstance()` creates a proxy at runtime for any interface
- **CGLIB proxy** — Spring uses CGLIB to proxy classes without interfaces
- **RMI (Remote Method Invocation)** — local stub is a remote proxy for a server object

---

## Common Mistakes

- Proxy doesn't implement same interface (breaks transparency)
- `HashMap` instead of `ConcurrentHashMap` (race conditions on concurrent put)
- No TTL on cache (stale data forever)
- Missing `volatile` on Virtual Proxy's lazy-init field (silent data race)
- Caching failure results (poisons cache; use `cache.remove(key)` on exception)

---

## When NOT to Use

- Object creation is cheap — Virtual Proxy adds complexity for no gain
- Caching is already handled by your framework (`@Cacheable`) — don't re-implement
- The behaviour you want to add changes the interface — use Adapter instead
