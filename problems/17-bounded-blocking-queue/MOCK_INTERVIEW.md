# Mock Interview Script — Bounded Blocking Queue

**For the interviewer.**

---

## Opening

> "Implement a thread-safe bounded blocking queue from scratch. `put(item)` should block if full, `take()` should block if empty. Multiple producer and consumer threads will use this simultaneously. Do NOT use any JDK BlockingQueue — implement it yourself using locks. Ask clarifying questions."

| If candidate asks | Answer |
|------------------|--------|
| Which lock type? | Candidate chooses — ReentrantLock is expected |
| Generic or typed? | Generic `T` is ideal; typed (Integer, String) is acceptable |
| Fairness required? | Not required for MVP — mention it as an extension |
| Capacity fixed? | Yes — set at construction, immutable |
| InterruptedException? | Propagate to caller — don't swallow |
| Non-blocking variants? | Secondary — `offer()` and `poll()` if time allows |

---

## Phase 2: Core Design — Watch For These Signals

**Signal 1 (most important): `while` not `if`**

If candidate writes:
```java
if (queue.size() == capacity) { notFull.await(); }
```
Stop them: > "What happens if `await()` returns spuriously — the OS wakes the thread for no reason?"

Expected correction: change `if` to `while`.

**Signal 2: Two Conditions, not one**

Good: `Condition notFull = lock.newCondition()` and `Condition notEmpty = lock.newCondition()` — producers and consumers in separate wait-sets.

Bad: One condition, using `signalAll()` — works but wasteful.

If candidate uses `synchronized` + `notifyAll()`:
> "What happens when a producer calls `notifyAll()` — who gets woken up?"
Expected: "All waiting threads — including other producers who can't proceed. With two Conditions I can wake ONLY consumers."

**Signal 3: `finally` block**

Good: `} finally { lock.unlock(); }` — lock released even on exception.

**Signal 4: Signal direction**

Explicitly verify: "`put()` signals `notEmpty`. `take()` signals `notFull`."

If reversed: point out silently and ask them to trace through a scenario.

---

## Extension Probe (~35 min)

> "Add `offer(item, timeout, unit)` — try to insert, but give up after the timeout."

Strong: `notFull.awaitNanos(remaining)` — returns remaining nanoseconds. Loop subtracts each wakeup's elapsed time from remaining. Returns false when remaining <= 0.

> "How would you make the queue fair — FIFO thread ordering for blocked threads?"

Strong: `new ReentrantLock(true)` — fair lock. Explains trade-off: ~30% throughput reduction.

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | Blocking put/take, fixed capacity, multi-producer/consumer, no busy-wait | |
| Entity Modeling | Two Conditions (notFull/notEmpty), ArrayDeque backing, single lock | |
| Design Patterns | Monitor Object / Producer-Consumer correctly implemented | |
| SOLID | N/A — this is a concurrency data structure problem | |
| Extensibility | offer/poll variants; fair lock; timed offer | |
| Concurrency | `while` loop, `finally`, correct signal direction, signal not signalAll | |
| Communication | Explained WHY two Conditions, WHY `while`, WHY `signal` not `signalAll` | |
| **TOTAL** | | **/21** |
