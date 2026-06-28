# Lessons Learned — Rate Limiter

---

## Mistake 1: Background Thread for Token Refill

**What happens:** Candidate creates a `ScheduledExecutorService` that refills all clients' token buckets every second.

**Why it's wrong:** Requires a thread per N clients or a sweeping thread that locks all buckets during refill. Doesn't scale. The "lazy refill" is strictly better.

**Fix:** Lazy refill — compute tokens earned based on elapsed time since last check:
```java
private void refill() {
    long now = System.nanoTime();
    double elapsed = (now - lastRefillTimeNanos) / 1_000_000_000.0;
    currentTokens = Math.min(capacity, currentTokens + elapsed * refillRate);
    lastRefillTimeNanos = now;
}
// Called inside tryConsume() — O(1), no background thread needed
```

**Gap dimension:** Entity Modeling (Dimension 2)

---

## Mistake 2: One Global Rate Limiter (Not Per-Client)

**What happens:** Single counter for all clients combined. First client to make requests blocks everyone else.

**Why it's wrong:** Rate limiting is per-client by definition. All requests from ClientA shouldn't affect ClientB's quota.

**Fix:**
```java
// Map of clientId → their own TokenBucket
private final Map<String, TokenBucket> buckets = new ConcurrentHashMap<>();
// Each client has independent state
```

**Gap dimension:** Requirements (Dimension 1) — missed in requirements phase

---

## Mistake 3: Race Condition on Token Decrement

**What happens:** `tryConsume()` reads token count, decides to allow, then decrements — but without synchronization. Two threads read the same count and both decide to allow.

**Fix:**
```java
// Option 1: synchronized
public synchronized boolean tryConsume() {
    refill();
    if (currentTokens >= 1.0) { currentTokens--; return true; }
    return false;
}

// Option 2: AtomicLong with CAS (lock-free, better throughput)
private final AtomicLong tokensScaled = new AtomicLong(capacity * 1000L);
public boolean tryConsume() {
    while (true) {
        long current = tokensScaled.get();
        if (current < 1000) return false;
        if (tokensScaled.compareAndSet(current, current - 1000)) return true;
    }
}
```

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 4: Return Only Boolean (Missing Metadata)

**What happens:** `isAllowed()` returns `boolean`. Client has no idea how many requests remain, or when to retry.

**Why it's wrong:** Good rate limiters return `X-RateLimit-Remaining`, `X-RateLimit-Limit`, `Retry-After` headers. Your `RateLimitResult` should carry this data.

**Fix:**
```java
class RateLimitResult {
    boolean allowed;
    int remainingTokens;
    long retryAfterMs; // 0 if allowed, milliseconds to wait if rejected
}
```

**Gap dimension:** Requirements / Entity Modeling (Dimensions 1, 2)

---

## Mistake 5: No Algorithm Comparison

**What happens:** Candidate picks fixed window without discussing trade-offs. Doesn't mention the 2× burst problem at window boundary.

**Why it's a gap:** Interviewers expect you to know why token bucket is preferred. Fixed window is simple but has a known flaw.

**The 2× burst problem:**
```
Window: 0–60s, limit 10 requests
Client sends 10 requests at t=59s (within window 1)
Client sends 10 requests at t=61s (within window 2)
In the span of 2 seconds: 20 requests passed — 2× the limit
Token bucket: impossible — bucket has max 10 tokens total
```

**Gap dimension:** Communication (Dimension 7)

---

## Problem-Specific Gotchas

- `ConcurrentHashMap.computeIfAbsent()` is the correct way to lazily create per-client bucket — avoids race on first request
- Token bucket capacity = burst allowance; refill rate = sustained throughput
- "Leaky bucket" is NOT the same as token bucket — leaky allows requests only at constant rate (no burst)
- Distributed rate limiting (Redis INCR + TTL) is a common follow-up — know the approach even if not implementing

---

## Self-Assessment Checklist

- [ ] Is token refill lazy (not background thread)?
- [ ] Is there a separate bucket per client (not global)?
- [ ] Is `tryConsume()` synchronized or using CAS?
- [ ] Does `isAllowed()` return remaining capacity + retry-after (not just boolean)?
- [ ] Did I compare token bucket vs fixed window trade-offs?
- [ ] Is `RateLimiter` an interface with multiple algorithm implementations?
- [ ] Can a new algorithm be added by adding one class?

---

## Remediation Targets

| Dimension | If you scored 0–1 | Go to |
|-----------|------------------|-------|
| Entity Modeling | Background thread, no per-client buckets | GAP_REMEDIATION.md#gap-2 |
| Design Patterns | No RateLimiter interface | GAP_REMEDIATION.md#gap-3 |
| Concurrency | Race on token decrement | GAP_REMEDIATION.md#gap-6 — shared state audit |
| Communication | No algorithm comparison | GAP_REMEDIATION.md#gap-7 — narrate reasoning |
