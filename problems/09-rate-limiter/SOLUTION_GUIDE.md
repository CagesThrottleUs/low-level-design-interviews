# Solution Guide — Rate Limiter

**Read after your attempt.**

---

## Entity Map

| Class/Interface | Type | Responsibility |
|----------------|------|---------------|
| `RateLimiter` | Interface | `isAllowed(clientId): RateLimitResult` |
| `TokenBucketRateLimiter` | Concrete | Per-client token bucket; thread-safe refill + consume |
| `SlidingWindowRateLimiter` | Concrete | Per-client timestamp queue; evicts old entries |
| `FixedWindowRateLimiter` | Concrete | Per-client counter + window reset timestamp |
| `TokenBucket` | Concrete | `tokens`, `lastRefillTime`, `capacity`, `refillRate`; `tryConsume()` |
| `RateLimitResult` | Concrete | `allowed: bool`, `remainingTokens: int`, `retryAfterMs: long` |
| `RateLimitConfig` | Concrete | Maps clientId → (limit, windowSeconds, algorithm) |
| `RateLimiterFactory` | Concrete | Creates correct `RateLimiter` based on algorithm config |

---

## Algorithm 1: Token Bucket (Primary)

**How it works:** Each client has a bucket with N tokens. Each request consumes one token. Tokens refill at a constant rate. If bucket is empty, request is rejected.

**Key insight:** Lazy refill — don't maintain a background thread. Compute how many tokens to add based on elapsed time since last refill.

```java
class TokenBucket {
    private final int capacity;         // max tokens (burst limit)
    private final double refillRate;    // tokens per second
    private double currentTokens;
    private long lastRefillTimeNanos;

    public TokenBucket(int capacity, double refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.currentTokens = capacity;
        this.lastRefillTimeNanos = System.nanoTime();
    }

    public synchronized boolean tryConsume() {
        refill();
        if (currentTokens >= 1.0) {
            currentTokens -= 1.0;
            return true;
        }
        return false;
    }

    private void refill() {
        long now = System.nanoTime();
        double elapsed = (now - lastRefillTimeNanos) / 1_000_000_000.0; // seconds
        double tokensToAdd = elapsed * refillRate;
        currentTokens = Math.min(capacity, currentTokens + tokensToAdd);
        lastRefillTimeNanos = now;
    }

    public synchronized int getRemainingTokens() {
        refill();
        return (int) currentTokens;
    }

    public synchronized long getRetryAfterMs() {
        // How long until 1 token is available?
        if (currentTokens >= 1.0) return 0;
        double deficit = 1.0 - currentTokens;
        return (long)(deficit / refillRate * 1000);
    }
}
```

**Why lazy refill:** Avoids a background thread per client. Correct because: "time elapsed × rate = tokens earned since last check". O(1) per request.

---

## RateLimiter Implementation

```java
class TokenBucketRateLimiter implements RateLimiter {
    private final Map<String, TokenBucket> buckets = new ConcurrentHashMap<>();
    private final RateLimitConfig config;

    public RateLimitResult isAllowed(String clientId) {
        TokenBucket bucket = buckets.computeIfAbsent(clientId, id -> {
            ClientConfig cfg = config.getConfig(clientId);
            return new TokenBucket(cfg.getCapacity(), cfg.getRefillRate());
        });

        boolean allowed = bucket.tryConsume();
        return new RateLimitResult(
            allowed,
            bucket.getRemainingTokens(),
            allowed ? 0 : bucket.getRetryAfterMs()
        );
    }
}
```

---

## Algorithm 2: Sliding Window Counter

**How it works:** Track timestamps of all requests in the last N seconds. On each request, evict old timestamps and check count.

```java
class SlidingWindowRateLimiter implements RateLimiter {
    private final Map<String, Deque<Long>> requestHistory = new ConcurrentHashMap<>();
    private final int limit;
    private final long windowMs;

    public synchronized RateLimitResult isAllowed(String clientId) {
        long now = System.currentTimeMillis();
        Deque<Long> history = requestHistory.computeIfAbsent(clientId, k -> new ArrayDeque<>());

        // Evict entries outside the window
        while (!history.isEmpty() && now - history.peekFirst() > windowMs) {
            history.pollFirst();
        }

        if (history.size() < limit) {
            history.addLast(now);
            return RateLimitResult.allowed(limit - history.size());
        }
        long oldestInWindow = history.peekFirst();
        long retryAfterMs = windowMs - (now - oldestInWindow);
        return RateLimitResult.rejected(retryAfterMs);
    }
}
```

**Trade-off vs token bucket:**
- Sliding window: no burst; exact count within any rolling window; O(n) memory per client
- Token bucket: allows burst up to capacity; O(1) memory per client; O(1) per check

---

## Algorithm 3: Fixed Window Counter (Simplest)

```java
class FixedWindowRateLimiter implements RateLimiter {
    private final Map<String, AtomicInteger> counters = new ConcurrentHashMap<>();
    private final Map<String, Long> windowStartTimes = new ConcurrentHashMap<>();
    private final int limit;
    private final long windowMs;

    public RateLimitResult isAllowed(String clientId) {
        long now = System.currentTimeMillis();
        windowStartTimes.merge(clientId, now, (existing, n) -> {
            return (now - existing > windowMs) ? now : existing;
        });

        // Reset counter if window expired
        AtomicInteger counter = counters.computeIfAbsent(clientId, k -> new AtomicInteger(0));
        if (now - windowStartTimes.get(clientId) > windowMs) {
            counter.set(0);
            windowStartTimes.put(clientId, now);
        }

        int count = counter.incrementAndGet();
        return count <= limit
            ? RateLimitResult.allowed(limit - count)
            : RateLimitResult.rejected(windowMs - (now - windowStartTimes.get(clientId)));
    }
}
```

**Known flaw:** A client can make `limit` requests at end of window N and `limit` requests at start of window N+1 — effectively 2× limit in a 2-second span. Token bucket and sliding window don't have this problem.

---

## Composite Key (Per-Endpoint Extension)

```java
// Instead of just clientId, use clientId + endpoint as key
public RateLimitResult isAllowed(String clientId, String endpoint) {
    String key = clientId + ":" + endpoint;
    return isAllowed(key); // same logic, composite key
}
```

Zero structural change — just change the key format.

---

## Concurrency: The Race Condition

```
Thread A: reads tokens = 1
Thread B: reads tokens = 1
Thread A: decrements → 0, returns allowed
Thread B: decrements → -1, returns allowed ← WRONG
```

**Fix:** `synchronized` on `tryConsume()` (already shown above). Alternative: `AtomicLong` with `compareAndSet`:

```java
class AtomicTokenBucket {
    private final AtomicLong tokensScaled; // tokens * 1000 for precision

    public boolean tryConsume() {
        while (true) {
            long current = tokensScaled.get();
            if (current < 1000) return false; // less than 1 token
            if (tokensScaled.compareAndSet(current, current - 1000)) return true;
            // CAS failed → another thread modified — retry
        }
    }
}
```

`compareAndSet` is lock-free — better throughput than `synchronized` under high contention.

---

## Extension Points

| "Now add..." | Change |
|-------------|--------|
| New algorithm | New class implementing `RateLimiter` |
| Per-endpoint limits | Composite key `clientId:endpoint` |
| Distributed (Redis) | New `RedisTokenBucketRateLimiter` using `HINCRBY` + `TTL` |
| Rate limit headers | Add to `RateLimitResult`; caller adds `X-RateLimit-*` headers |

---

## What Strong Candidates Do Differently

- Lazy refill (no background thread) explained clearly
- Compare token bucket vs fixed window trade-offs without being asked
- `ConcurrentHashMap.computeIfAbsent` for bucket creation (avoids race on first request)
- `compareAndSet` or `synchronized` on `tryConsume()` with explicit reasoning
- `RateLimitResult` includes remaining tokens AND retry-after (not just boolean)

## What Average Candidates Miss

- Background thread for refill (complex, unnecessary)
- No per-client limits — one global counter for all clients
- No thread safety on token decrement
- Return only boolean — missing remaining capacity and retry-after
- No comparison between algorithms — pick one without explaining why
