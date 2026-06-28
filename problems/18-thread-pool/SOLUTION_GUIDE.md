# Solution Guide — Custom Thread Pool

**Read after your attempt.**

---

## Entity Map

| Class | Type | Responsibility |
|-------|------|---------------|
| `ThreadPool` | Outer manager | Owns state, task queue, worker list; submit/shutdown API |
| `PoolState` | Enum | RUNNING → SHUTDOWN → TERMINATED |
| `WorkerThread` | Inner Thread subclass | Loop: poll → execute → check state; wraps run() in try-catch |
| `LinkedBlockingQueue<Runnable>` | Task queue | Decouples submitters from workers; provides blocking take/poll |
| `volatile PoolState state` | State field | Workers read this; submitters write it on shutdown — must be volatile |
| `AtomicInteger completedCount` | Counter | Thread-safe increment without a lock |
| `FutureTask<T>` | JDK wrapper | Wraps Callable into Runnable; provides Future interface to caller |

---

## Full Implementation

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadPool {

    private enum PoolState { RUNNING, SHUTDOWN, TERMINATED }

    private volatile PoolState state = PoolState.RUNNING;

    private final BlockingQueue<Runnable>  taskQueue;
    private final List<WorkerThread>       workers;
    private final int                      poolSize;
    private final AtomicInteger            completedCount = new AtomicInteger(0);

    public ThreadPool(int poolSize) {
        this(poolSize, Integer.MAX_VALUE);
    }

    public ThreadPool(int poolSize, int queueCapacity) {
        if (poolSize <= 0) throw new IllegalArgumentException("Pool size must be > 0");
        this.poolSize  = poolSize;
        this.taskQueue = new LinkedBlockingQueue<>(queueCapacity);
        this.workers   = new ArrayList<>(poolSize);

        // Create and start all workers eagerly
        for (int i = 0; i < poolSize; i++) {
            WorkerThread w = new WorkerThread("pool-worker-" + i);
            workers.add(w);
            w.start();
        }
    }

    /** Submit a Runnable. Throws RejectedExecutionException if pool is shut down. */
    public void submit(Runnable task) {
        Objects.requireNonNull(task, "Task must not be null");
        if (state != PoolState.RUNNING) {
            throw new RejectedExecutionException("ThreadPool is shutting down");
        }
        // offer() preferred over add() — returns false on full queue
        if (!taskQueue.offer(task)) {
            throw new RejectedExecutionException("Task queue is full");
        }
    }

    /** Submit a Callable — returns Future<T> for result retrieval. */
    public <T> Future<T> submit(Callable<T> callable) {
        Objects.requireNonNull(callable, "Callable must not be null");
        if (state != PoolState.RUNNING) {
            throw new RejectedExecutionException("ThreadPool is shutting down");
        }
        FutureTask<T> future = new FutureTask<>(callable);
        if (!taskQueue.offer(future)) {
            throw new RejectedExecutionException("Task queue is full");
        }
        return future;
    }

    /**
     * Graceful shutdown: stop accepting new tasks; existing tasks complete.
     * Workers drain the queue before terminating.
     */
    public void shutdown() {
        state = PoolState.SHUTDOWN;
        // Interrupt workers blocked on queue.poll(timeout) so they re-check state
        for (WorkerThread w : workers) {
            w.interrupt();
        }
    }

    /**
     * Forceful shutdown: interrupt workers immediately.
     * Returns tasks that were never started.
     */
    public List<Runnable> shutdownNow() {
        state = PoolState.SHUTDOWN;
        List<Runnable> unexecuted = new ArrayList<>();
        taskQueue.drainTo(unexecuted);      // atomically drain remaining tasks
        for (WorkerThread w : workers) {
            w.interrupt();                  // interrupt running tasks too
        }
        return unexecuted;
    }

    /** Block until all workers have terminated. */
    public void awaitTermination() throws InterruptedException {
        for (WorkerThread w : workers) {
            w.join();                        // main thread waits for each worker
        }
        state = PoolState.TERMINATED;
    }

    public boolean isShutdown()    { return state != PoolState.RUNNING; }
    public boolean isTerminated()  { return state == PoolState.TERMINATED; }
    public int getCompletedCount() { return completedCount.get(); }
    public int getQueueSize()      { return taskQueue.size(); }

    // ── Worker thread ────────────────────────────────────────────────────────

    private class WorkerThread extends Thread {

        WorkerThread(String name) {
            super(name);
            setDaemon(false); // non-daemon: JVM won't exit while workers alive
        }

        @Override
        public void run() {
            // Workers keep running until SHUTDOWN AND queue is fully drained
            while (state == PoolState.RUNNING || !taskQueue.isEmpty()) {
                try {
                    // poll with timeout — wakes up periodically to re-check state
                    // This avoids being stuck on take() forever after shutdown()
                    Runnable task = taskQueue.poll(100, TimeUnit.MILLISECONDS);
                    if (task != null) {
                        try {
                            task.run();
                            completedCount.incrementAndGet();
                        } catch (Exception e) {
                            // Task threw — log but DO NOT rethrow
                            // Worker must survive task exceptions
                            System.err.println(getName() + ": task exception: " + e.getMessage());
                        }
                    }
                } catch (InterruptedException e) {
                    // Interrupted by shutdown() — restore flag and re-check loop condition
                    Thread.currentThread().interrupt();
                    if (state == PoolState.SHUTDOWN && taskQueue.isEmpty()) {
                        break; // clean exit
                    }
                    Thread.interrupted(); // clear flag to allow next poll()
                }
            }
        }
    }
}
```

---

## Usage Example

```java
public class ThreadPoolDemo {
    public static void main(String[] args) throws Exception {
        ThreadPool pool = new ThreadPool(4, 100);

        // Submit Runnables
        for (int i = 0; i < 10; i++) {
            final int id = i;
            pool.submit(() -> {
                System.out.println(Thread.currentThread().getName()
                    + " executing task " + id);
                try { Thread.sleep(200); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            });
        }

        // Submit a Callable with a result
        Future<Integer> future = pool.submit(() -> {
            Thread.sleep(100);
            return 42;
        });
        System.out.println("Callable result: " + future.get());

        pool.shutdown();
        pool.awaitTermination();
        System.out.println("Completed: " + pool.getCompletedCount());
    }
}
```

---

## The Four Critical Design Decisions

### 1. `volatile` on Pool State

Workers on multiple CPUs read `state` on every loop iteration. Without `volatile`, each CPU can cache a stale copy. Worker sees RUNNING forever even after `shutdown()` was called.

```java
private volatile PoolState state; // visibility guarantee across CPUs
```

### 2. `poll(timeout)` Not `take()` in Workers

If workers use `take()`, they block indefinitely. After `shutdown()` is called, there's no way to wake them unless `shutdownNow()` interrupts. With `poll(100ms)`, workers wake automatically every 100ms to re-check the loop condition.

```java
Runnable task = taskQueue.poll(100, TimeUnit.MILLISECONDS);
if (task != null) { /* execute */ }
// If null: loop re-checks state — exits cleanly on SHUTDOWN + empty queue
```

### 3. Worker Must Not Die on Task Exception

```java
try {
    task.run();
} catch (Exception e) {
    // Log and continue — worker stays alive
    System.err.println("Task threw: " + e.getMessage());
}
// If task.run() throws and we don't catch it, the worker thread dies
// Pool silently loses a thread — remaining tasks run on fewer workers
```

### 4. Graceful Shutdown Loop Condition

```java
while (state == PoolState.RUNNING || !taskQueue.isEmpty()) { ... }
```

Both conditions needed:
- `state == RUNNING`: keep going while accepting new tasks
- `!taskQueue.isEmpty()`: drain remaining tasks even after shutdown

If only `state == RUNNING`: workers exit immediately on shutdown, leaving queued tasks unexecuted.

---

## Thread Naming + Daemon Setting

```java
WorkerThread(String name) {
    super(name); // thread name in stack traces — essential for debugging
    setDaemon(false); // non-daemon: JVM won't exit while workers are alive
}
```

`daemon = true` would let JVM exit while workers are running — correct for background logger threads, wrong for a work pool where tasks must complete.

---

## Extension Points

| "Now add..." | Change |
|-------------|--------|
| Priority queue | `PriorityBlockingQueue<PrioritizedTask>` where task implements `Comparable` |
| Dynamic resizing | Track `coreSize` vs `maxSize`; spawn new workers when queue grows; `keepAliveTime` for idle workers |
| Rejection policies | `RejectionPolicy` interface: `AbortPolicy`, `CallerRunsPolicy`, `DiscardPolicy` |
| Task cancellation | `FutureTask.cancel(mayInterruptIfRunning)` already supported |
| Metrics | `AtomicLong submitCount, rejectedCount, taskDurationNanos` |
| Named thread factory | `ThreadFactory` interface — `new ThreadFactory() { Thread newThread(Runnable r) { ... } }` |

---

## What Strong Candidates Do Differently

- `volatile PoolState` with enum (not boolean `isShutdown`) — enables TERMINATED state
- `poll(timeout)` not `take()` — explains the shutdown problem with `take()`
- Worker try-catch around `task.run()` — explicitly states "pool must survive task exceptions"
- Loop condition checks BOTH state AND queue emptiness
- `FutureTask` for Callable — doesn't reinvent it
- Non-daemon workers — explains JVM exit behavior

## What Average Candidates Miss

- Worker dying on task RuntimeException — pool silently shrinks
- `take()` instead of `poll(timeout)` — workers stuck after shutdown
- Missing `volatile` on state — stale cached state across CPUs
- Loop exits on SHUTDOWN before draining queue — unexecuted tasks
- Creating workers lazily on submit (not eagerly) — "fixed pool" means pre-created
