# Custom Thread Pool

**Difficulty:** Advanced
**Category:** Concurrency — Thread Lifecycle + State Machines
**Time Box:** 60 min (discussion + implementation)
**Key Patterns:** Thread Pool, State Machine (pool lifecycle), Producer-Consumer (task queue)
**Asked at:** Google, Amazon, Meta, Goldman Sachs, Bloomberg

---

## Problem Statement

Implement a fixed-size thread pool from scratch. The pool pre-creates a configurable number of worker threads that wait for submitted tasks. `submit(Runnable)` adds a task to the queue; worker threads pick up and execute tasks. The pool supports graceful shutdown: stop accepting new tasks, finish executing all submitted tasks, then terminate workers.

---

## Actors

| Actor | Description |
|-------|-------------|
| Client / Caller | Calls `submit(task)` and eventually `shutdown()` |
| ThreadPool | Manages worker threads + task queue + lifecycle state |
| WorkerThread | Loop: pull task from queue → execute → repeat until shutdown |
| Task | Any `Runnable` or `Callable<T>` |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: Fixed number of worker threads created at construction (not on demand)
- FR2: `submit(Runnable task)` — adds task to queue; non-blocking (throws `RejectedExecutionException` if pool is shut down)
- FR3: Worker threads continuously pull tasks from queue and execute them
- FR4: Worker thread must NOT die if a task throws a runtime exception — pool must stay alive
- FR5: `shutdown()` — stop accepting new tasks; allow all submitted tasks to complete; terminate workers after queue is drained
- FR6: `awaitTermination()` — block caller until all workers have terminated

**Secondary (implement if time allows):**
- FR7: `shutdownNow()` — interrupt workers immediately; return unexecuted tasks
- FR8: `submit(Callable<T>): Future<T>` — return a future for result retrieval
- FR9: Task priority queue — higher priority tasks execute first
- FR10: Thread factory — configurable thread creation (daemon threads, custom names)

---

## Non-Functional Requirements

- **Reliability:** Worker must survive task exceptions — `RuntimeException` in `task.run()` must not kill the thread
- **State safety:** `volatile` on pool state — workers and submitters see consistent state
- **No busy-waiting:** Workers block on `queue.poll(timeout)` not spin in a loop
- **Graceful shutdown:** Tasks submitted before `shutdown()` must complete

---

## Constraints and Assumptions

- Fixed pool size (not min/max/core like `ThreadPoolExecutor`)
- `LinkedBlockingQueue<Runnable>` as task queue (using JDK queue IS acceptable here — the pool itself, not the queue, is what's being implemented)
- `volatile` enum for pool state (RUNNING, SHUTDOWN, TERMINATED)
- Out of scope: dynamic pool sizing, work-stealing, rejection policies (beyond throw)

---

## Good Clarifying Questions to Ask

1. Fixed pool size or dynamic (min/max core threads)?
2. Bounded or unbounded task queue?
3. If task throws, should worker die or recover?
4. Graceful shutdown (drain queue) or immediate (interrupt all)?
5. `Future<T>` support for Callable tasks?
6. Thread naming / daemon threads?

---

## Key Design Difference from Bounded Blocking Queue

In the Bounded Blocking Queue problem, you build the queue itself using `ReentrantLock` + `Condition`. In Thread Pool, you USE a `BlockingQueue` as an off-the-shelf component — the focus here is the pool's worker lifecycle, state machine, and shutdown sequence, not re-implementing the queue primitive.

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **ThreadPool** — owns task queue, worker list, pool state; manages lifecycle
- **`volatile PoolState state`** — RUNNING / SHUTDOWN / TERMINATED enum; `volatile` for cross-thread visibility without lock
- **`BlockingQueue<Runnable> taskQueue`** — `LinkedBlockingQueue` (unbounded) or bounded; decouples submitters from workers
- **WorkerThread** (inner class, extends Thread) — runs loop: `queue.poll(100ms)` → execute task → check state → repeat; wraps `task.run()` in try-catch
- **`AtomicInteger completedCount`** — tracks completed tasks without a lock
- **`FutureTask<T>`** — JDK class wrapping `Callable<T>` into a `Runnable`; returned to caller on `submit(Callable)`

</details>
