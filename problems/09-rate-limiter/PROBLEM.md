# Rate Limiter

**Difficulty:** Intermediate
**Category:** Algorithm + Strategy
**Time Box:** 45 min (discussion) / 90 min (machine coding)
**Key Patterns:** Strategy (algorithm selection), Decorator (layered limiting)
**Asked at:** Google, Meta, Cloudflare, Stripe, Amazon, Uber

---

## Problem Statement

Design a rate limiter that restricts how many requests a client can make to an API within a time window. The rate limiter should be configurable (different limits per client/endpoint), support multiple algorithms (token bucket, sliding window, fixed window counter), and be thread-safe.

---

## Actors

| Actor | Description |
|-------|-------------|
| API Client | Makes requests; gets allowed or rejected |
| Rate Limiter | Intercepts requests; applies configured limit; returns allow/deny |
| Configuration | Maps client ids / endpoints to rate limit rules |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: `isAllowed(clientId)` — returns true if request is within limit, false if rate-limited
- FR2: Support token bucket algorithm — N tokens refilled per second, each request consumes 1
- FR3: Different limits configurable per client (e.g., free tier: 10 req/min, paid: 1000 req/min)
- FR4: Thread-safe — multiple threads calling isAllowed() simultaneously
- FR5: Return remaining tokens / time-to-reset as metadata

**Secondary (implement if time allows):**
- FR6: Sliding window counter algorithm (smoother than fixed window)
- FR7: Fixed window counter algorithm (simplest)
- FR8: Per-endpoint limits (not just per-client)
- FR9: Distributed rate limiting across multiple nodes (discuss only)
- FR10: Rate limiter as a Decorator over an API handler

---

## Non-Functional Requirements

- **Correctness:** Never allow more than N requests in the time window
- **Concurrency:** Multiple threads calling isAllowed() on the same client concurrently
- **Latency:** Rate limiter check must be sub-millisecond
- **Precision:** Token bucket is more precise than fixed window (no burst at boundary)

---

## Constraints and Assumptions

- In-memory; not distributed for MVP
- Time measured in seconds
- Client identified by a string ID (IP address, API key, user ID)
- Algorithms: token bucket required; sliding window is bonus
- Out of scope: Redis-backed distributed limiter, leaky bucket

---

## Good Clarifying Questions to Ask

1. What algorithm — token bucket, sliding window, fixed window?
2. Per-client limits or global limits or both?
3. Per-endpoint limits too?
4. Thread-safe for single machine, or distributed across multiple nodes?
5. What to return — boolean, or also remaining capacity + reset time?
6. Should different clients have different rate limits?
7. What happens when limit is hit — reject immediately or queue?

---

## Algorithm Comparison (know before designing)

| Algorithm | Burst handling | Boundary spike | Complexity | Memory |
|-----------|--------------|----------------|-----------|--------|
| Fixed window | Allows full burst at window boundary | Yes (2× burst possible) | Simple | O(1) per client |
| Sliding window | Smooth, no boundary spike | No | Moderate | O(window/bucket) |
| Token bucket | Allows controlled burst up to bucket size | No | Simple | O(1) per client |
| Leaky bucket | No burst — strict rate | No | Simple | O(1) per client |

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **RateLimiter** — interface: `isAllowed(clientId): RateLimitResult`
- **TokenBucketRateLimiter** — implements RateLimiter; per-client token bucket
- **SlidingWindowRateLimiter** — implements RateLimiter; tracks timestamps in sliding window
- **FixedWindowRateLimiter** — implements RateLimiter; counter per time window
- **TokenBucket** — holds tokens, lastRefillTime; `tryConsume()` method
- **RateLimitResult** — allowed: boolean, remainingTokens: int, retryAfterMs: long
- **RateLimitConfig** — maps clientId → (limit, windowSeconds)
- **RateLimiterFactory** — creates correct limiter based on config algorithm type

</details>
