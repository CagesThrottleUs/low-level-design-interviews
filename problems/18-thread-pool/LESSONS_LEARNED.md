# Lessons Learned — Custom Thread Pool

---

## Mistake 1: Worker Dies on Task Exception

**What happens:**
```java
public void run() {
    while (...) {
        Runnable task = taskQueue.poll(100, TimeUnit.MILLISECONDS);
        if (task != null) {
            task.run(); // RuntimeException propagates → thread exits
        }
    }
}
```

**Why it's wrong:** If `task.run()` throws `RuntimeException`, it propagates out of `run()`, the thread terminates. Pool silently has fewer workers. After 4 bad tasks on a 4-thread pool, no workers remain — tasks queue up forever.

**Fix:**
```java
try {
    task.run();
    completedCount.incrementAndGet();
} catch (Exception e) {
    System.err.println(getName() + ": task exception: " + e.getMessage());
    // Worker continues loop — pool stays healthy
}
```

**Gap dimension:** Concurrency / Requirements (Dimensions 6, 1)

---

## Mistake 2: Using `take()` Instead of `poll(timeout)`

**What happens:** `Runnable task = taskQueue.take()` — blocks indefinitely.

**Why it's wrong:** After `shutdown()`, workers blocked on `take()` stay blocked. Only `shutdownNow()` + `interrupt()` can wake them. Graceful shutdown hangs forever.

**Fix:** `taskQueue.poll(100, TimeUnit.MILLISECONDS)` — wakes automatically every 100ms. Workers re-check the loop condition and exit cleanly:
```java
// Exits when: state == SHUTDOWN && queue.isEmpty()
while (state == PoolState.RUNNING || !taskQueue.isEmpty()) {
    Runnable task = taskQueue.poll(100, TimeUnit.MILLISECONDS);
    // null if timeout expired — loop re-checks condition
}
```

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 3: Missing `volatile` on Pool State

**What happens:** `private PoolState state = PoolState.RUNNING` — no `volatile`.

**Why it's wrong:** JVM can cache `state` in a CPU register. Worker on CPU-1 reads cached RUNNING even after the main thread on CPU-0 wrote SHUTDOWN. Worker loops forever.

**Fix:** `private volatile PoolState state` — volatile write in `shutdown()` creates a happens-before relationship; all subsequent volatile reads see SHUTDOWN.

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 4: Shutdown Loop Abandons Queued Tasks

**What happens:** `while (state == PoolState.RUNNING)` — loop exits immediately when state becomes SHUTDOWN. 50 queued tasks are abandoned.

**Why it's wrong:** "Graceful shutdown" means drain the queue. Tasks submitted before `shutdown()` must complete.

**Fix:** Loop exits only when BOTH: state is SHUTDOWN AND queue is empty:
```java
while (state == PoolState.RUNNING || !taskQueue.isEmpty()) { ... }
```

**Gap dimension:** Requirements (Dimension 1)

---

## Mistake 5: Creating Workers Lazily on Submit

**What happens:** No workers created in constructor. Each `submit()` creates a new thread up to `poolSize`.

**Why it's wrong:** This is a cached thread pool, not a fixed thread pool. "Fixed" means all `poolSize` threads exist from the start — no latency spike on the first N submissions. Also: lazy creation needs careful thread-safe implementation.

**Fix:** Create and start all workers in the constructor:
```java
for (int i = 0; i < poolSize; i++) {
    WorkerThread w = new WorkerThread("pool-worker-" + i);
    workers.add(w);
    w.start(); // all threads ready before first submit()
}
```

**Gap dimension:** Requirements (Dimension 1)

---

## Mistake 6: Daemon Workers

**What happens:** `setDaemon(true)` on worker threads.

**Why it's wrong:** Daemon threads are killed when the last non-daemon thread exits. If the main thread finishes but the pool still has tasks, all workers are killed mid-execution — data loss, corrupted state.

**Fix:** `setDaemon(false)` (default) — workers are non-daemon. JVM waits for them to finish before exiting. `awaitTermination()` lets the main thread explicitly wait.

**Gap dimension:** Concurrency (Dimension 6)

---

## Self-Assessment Checklist

- [ ] Is `task.run()` wrapped in try-catch(Exception)?
- [ ] Do workers use `poll(timeout)` not `take()`?
- [ ] Is pool state `volatile`?
- [ ] Does the worker loop drain queue after SHUTDOWN (`|| !taskQueue.isEmpty()`)?
- [ ] Are workers created eagerly in the constructor?
- [ ] Are workers non-daemon (`setDaemon(false)`)?
- [ ] Does `awaitTermination()` use `thread.join()` (not sleep or poll)?

---

## PoolState Transitions

```
RUNNING → (shutdown() called) → SHUTDOWN → (all workers exit) → TERMINATED
```

- RUNNING: accepting and executing tasks
- SHUTDOWN: no new tasks; draining queue; workers finishing
- TERMINATED: all workers exited; `awaitTermination()` returns

---

## Remediation Targets

| Dimension | If scored 0–1 | Go to |
|-----------|--------------|-------|
| Concurrency | Missing volatile, take() not poll(), no try-catch in worker | GAP_REMEDIATION.md#gap-6 |
| Requirements | Worker dies on exception, queue abandoned on shutdown | GAP_REMEDIATION.md#gap-1 |
| Communication | Couldn't explain volatile, daemon thread, poll vs take | GAP_REMEDIATION.md#gap-7 |
