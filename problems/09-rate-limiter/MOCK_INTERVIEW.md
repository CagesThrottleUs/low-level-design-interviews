# Mock Interview Script — Rate Limiter

**For the interviewer.**

---

## Opening

> "Design a rate limiter for an API. It should limit how many requests a client can make in a given time window. Ask any clarifying questions before designing."

| If candidate asks | Answer |
|------------------|--------|
| Which algorithm? | Candidate should propose; token bucket is preferred |
| Per-client or global? | Per-client (each API key has its own limit) |
| Different limits per client? | Yes — free tier 100 req/min, paid tier 10000 req/min |
| Thread-safe? | Yes — multiple threads will call isAllowed() concurrently |
| Return remaining capacity? | Yes — include remaining tokens and retry-after in response |
| Distributed? | Single-machine for MVP; discuss distributed as extension |
| Per-endpoint limits too? | Secondary — focus on per-client first |

---

## Phase 2: Core Design — What to Watch For

**Key signal 1:** Algorithm understanding
- Can candidate explain token bucket vs fixed window trade-offs?
- Token bucket: allows burst up to bucket size; smooth refill
- Fixed window: simple but "burst at boundary" problem (2× burst in 2 seconds at window crossing)

**Key signal 2:** O(1) check per request
- `isAllowed()` should not iterate or scan history — must be O(1) per client

**Key signal 3:** Thread safety on token bucket
- Two threads calling `isAllowed()` simultaneously on same client → race on token count

If candidate only describes one algorithm without comparing:
> "What are the trade-offs between fixed window and token bucket? When would each be preferred?"

If no interface extracted:
> "If we need to run token bucket for some clients and sliding window for others, how does your design support that?"

---

## Extension Probe (~35 min)

> "Now add per-endpoint limits — same client has a higher limit on GET endpoints than POST endpoints."

Strong: Rate limiter keyed on `(clientId, endpoint)` not just `clientId`. Or composite key. `RateLimitConfig` holds a map of these composite keys. No structural change to `TokenBucket`.

> "How would you make this distributed — rate limiting across 10 server nodes?"

Strong: Mentions Redis-backed token bucket (`INCR`, `EXPIRE`). Or approximate approaches: each node tracks partial budget. Exact distributed rate limiting requires atomic Redis operations or consensus.

---

## Concurrency Probe (~40 min, if not raised)

> "Two threads for the same client call isAllowed() simultaneously. Walk me through the race condition."

Strong: Both threads read `tokens = 5`, both decide to allow, both decrement to 4. Net result: two requests allowed, tokens only decremented once. Fix: `synchronized` on `tryConsume()` or `AtomicLong` with `compareAndSet`.

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | Algorithm choice with justification, per-client limits, thread safety, remaining capacity | |
| Entity Modeling | TokenBucket as separate class, RateLimiter interface, RateLimitConfig, RateLimitResult | |
| Design Patterns | Strategy (algorithm selection), Factory (create correct limiter per config) | |
| SOLID | OCP — new algorithm = new class | |
| Extensibility | New algorithm or composite key absorbed cleanly | |
| Concurrency | Race on token count → synchronized tryConsume or AtomicLong | |
| Communication | Explained WHY token bucket, WHY O(1) matters, compared algorithms | |
| **TOTAL** | | **/21** |
