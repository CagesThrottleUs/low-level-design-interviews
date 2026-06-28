# Mock Interview Script — Custom Thread Pool

**For the interviewer.**

---

## Opening

> "Implement a fixed-size thread pool from scratch. Worker threads should wait for submitted tasks, execute them, and loop for more. Implement graceful shutdown. Ask clarifying questions first."

| If candidate asks | Answer |
|------------------|--------|
| Fixed or dynamic pool size? | Fixed — all threads created at construction |
| Bounded or unbounded queue? | Unbounded for MVP; bounded is secondary |
| Task throws exception — worker dies? | No — worker must survive and continue |
| Graceful or immediate shutdown? | Graceful (drain queue) for MVP; immediate as secondary |
| `Future<T>` for Callable? | Secondary — Runnable only for MVP |
| Thread naming? | Yes — name each worker thread |

---

## Phase 2: Core Design — What to Watch For

**Signal 1: `volatile` on pool state**

Good: `private volatile PoolState state` — explains workers on different CPUs need to see shutdown immediately.

Bad: `private boolean isShutdown` without `volatile` — workers may see stale cached value.

**Signal 2: `poll(timeout)` not `take()` in workers**

Good: "If workers use `take()`, they block indefinitely — there's no way to wake them after shutdown() except interrupt. Using `poll(100ms)` lets workers re-check state periodically."

Bad: `taskQueue.take()` with no interrupt handling — pool hangs forever after shutdown.

**Signal 3: Worker survives task exceptions**

Good: `task.run()` wrapped in try-catch(Exception). Explicitly states: "If the worker thread dies on a RuntimeException, the pool silently shrinks. This must be caught."

Bad: No try-catch around `task.run()` — fatal flaw.

**Signal 4: Shutdown loop drains queue**

Good: `while (state == RUNNING || !taskQueue.isEmpty())` — drains remaining tasks after shutdown.

Bad: `while (state == RUNNING)` — exits immediately on shutdown, abandoning queued tasks.

If candidate misses signal 4:
> "You call `shutdown()`, then `awaitTermination()`. There are 50 tasks still in the queue. What happens to them?"

---

## Extension Probe (~40 min)

> "A submitted task calls `future.cancel(true)` after 5 seconds. How does your pool handle this?"

Strong: `submit(Callable)` returns `FutureTask<T>`. `cancel(true)` calls `Thread.interrupt()` on the executing worker. Worker's `task.run()` (which is `futureTask.run()`) handles interruption internally.

> "How would you size this pool for an I/O-heavy workload vs a CPU-bound workload?"

Strong: CPU-bound: `Runtime.getRuntime().availableProcessors()` or `+1`. I/O-bound: `cores × (1 + wait_time / compute_time)` — threads spend most time waiting, so more threads can be useful.

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | Worker survives exceptions, graceful shutdown drains queue, fixed pool size | |
| Entity Modeling | PoolState enum, WorkerThread inner class, volatile state, AtomicInteger | |
| Design Patterns | State machine (RUNNING/SHUTDOWN/TERMINATED), Producer-Consumer on task queue | |
| SOLID | SRP: pool manages lifecycle; worker executes tasks — separated | |
| Extensibility | FutureTask for Callable; priority queue; rejection policies | |
| Concurrency | volatile state, poll(timeout), try-catch in worker, correct shutdown loop | |
| Communication | Explained volatile reason, why poll not take, why non-daemon | |
| **TOTAL** | | **/21** |
