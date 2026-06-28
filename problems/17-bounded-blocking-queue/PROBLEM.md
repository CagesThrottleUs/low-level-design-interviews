# Bounded Blocking Queue (Producer-Consumer)

**Difficulty:** Intermediate-Advanced
**Category:** Concurrency — Locks + Condition Variables
**Time Box:** 45 min (discussion + implementation)
**Key Patterns:** Producer-Consumer, Monitor Object
**Asked at:** Google, Amazon, Meta, Microsoft, Bloomberg

---

## Problem Statement

Implement a thread-safe bounded blocking queue from scratch. The queue has a fixed capacity. `put(item)` blocks if the queue is full until space becomes available. `take()` blocks if the queue is empty until an item is available. The implementation must support multiple producer threads and multiple consumer threads simultaneously.

**Critical rule:** Do NOT wrap `java.util.concurrent.LinkedBlockingQueue` or any other JDK `BlockingQueue`. The interviewer wants to see you build the primitive using `ReentrantLock` and `Condition` variables.

---

## Actors

| Actor | Description |
|-------|-------------|
| Producer threads | Call `put(item)` — block when queue is full |
| Consumer threads | Call `take()` — block when queue is empty |
| BoundedBlockingQueue | Coordinates producers and consumers; enforces capacity |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: `put(T item)` — inserts item; blocks if queue full until space available
- FR2: `take(): T` — removes and returns head; blocks if empty until item available
- FR3: Fixed capacity set at construction; cannot change
- FR4: Thread-safe — N producers and M consumers run concurrently without data corruption
- FR5: No busy-waiting — sleeping threads must not spin (CPU should be idle while blocked)
- FR6: `size(): int` — returns current element count (thread-safe)

**Secondary (implement if time allows):**
- FR7: `offer(T item): boolean` — non-blocking; returns false immediately if full
- FR8: `poll(): T` — non-blocking; returns null immediately if empty
- FR9: `offer(T, timeout, unit): boolean` — timed blocking offer
- FR10: `drainTo(Collection<T>)` — atomically drain all items

---

## Non-Functional Requirements

- **No busy-waiting:** Blocked threads must sleep (using `Condition.await()`)
- **Correctness:** Spurious wakeups must be handled (`while` not `if` on condition check)
- **Signal efficiency:** `signal()` preferred over `signalAll()` when one waiter can proceed
- **Fairness:** Optional — `new ReentrantLock(true)` for FIFO thread ordering

---

## Constraints and Assumptions

- Generic type `T` for the queue element type
- Capacity >= 1 enforced at construction
- `InterruptedException` propagated to caller — caller decides interrupt policy
- `ArrayDeque<T>` preferred over `LinkedList<T>` as backing store (avoids GC churn)
- Do NOT use: `java.util.concurrent.BlockingQueue`, `synchronized` + `wait/notify` (use `ReentrantLock` + `Condition`)

---

## Good Clarifying Questions to Ask

1. Fixed capacity or dynamic? (fixed)
2. Multiple producers and consumers simultaneously? (yes)
3. Should blocked threads spin or sleep? (sleep — no busy-waiting)
4. FIFO ordering of blocked threads? (optional — fair lock)
5. `InterruptedException` — propagate or swallow? (propagate)
6. Non-blocking variants needed (`offer`, `poll`)? (secondary)

---

## Why ReentrantLock + Condition, Not synchronized + wait/notify

| | `synchronized` + `wait/notify` | `ReentrantLock` + `Condition` |
|---|---|---|
| Condition queues | ONE per object | Multiple per lock |
| Precision waking | `notify()` wakes random thread | `condition.signal()` wakes from specific queue |
| Use case | One type of waiter | Producers wait on `notFull`, consumers on `notEmpty` |
| Problem with one queue | `notify()` might wake producer when consumer is needed | Not possible with two Conditions |
| Fair ordering | Not available | `new ReentrantLock(true)` |

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **BoundedBlockingQueue<T>** — owns all state
- **`Deque<T> queue`** — `ArrayDeque` as backing store; mutated under lock
- **`ReentrantLock lock`** — single mutex protecting all state
- **`Condition notFull`** — producers `await()` here when `size == capacity`; consumers `signal()` here after `take()`
- **`Condition notEmpty`** — consumers `await()` here when `size == 0`; producers `signal()` here after `put()`
- **`int capacity`** — immutable upper bound

The two-Condition design is the core insight: it lets producers and consumers sleep in SEPARATE queues, so waking one producer doesn't accidentally wake a consumer (and vice versa).

</details>
