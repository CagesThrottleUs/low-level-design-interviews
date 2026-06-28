# Java Concurrency — Complete Reference

---

## Foundation: The Java Memory Model (JMM)

Before any primitive makes sense, understand the three properties every primitive trades in:

| Property | Meaning |
| --- | --- |
| **Visibility** | A write by one thread is seen by other threads |
| **Atomicity** | An operation completes without partial intermediate state visible to others |
| **Ordering** | Instructions execute in the expected sequence (compilers/CPUs reorder for speed) |

Every Java concurrency primitive gives a different subset of these three. The decision of which primitive to use is always: "which properties do I need here?"

### Happens-Before (HB)

"A happens-before B" is a formal guarantee — not a wall-clock statement. It means: every write done before A is guaranteed visible to B. Created by:

| What creates HB | Rule |
| --- | --- |
| `volatile` write → subsequent read | Write to `volatile x` HB any read of `x` that follows |
| `synchronized` unlock → next lock | Exiting a block HB any thread that next enters the same monitor |
| `Thread.start()` | Actions before `start()` HB any action inside the new thread |
| `Thread.join()` | All thread actions HB the return from `join()` |
| Static initializer | Class init HB first use of the class by any thread |

---

## Primitive Map — One-Line Decision

```
volatile          → visibility only, single writer, no compound ops
AtomicInteger     → atomic compound op (increment, CAS), multiple writers
LongAdder         → high-contention counter, exact live value not needed
AtomicReference   → lock-free object swap / publish
synchronized      → simple critical section, low contention, need wait/notify
ReentrantLock     → need tryLock / timeout / multiple conditions / fairness
ReadWriteLock     → many reads, rare writes
StampedLock       → reads dominate, writes ultra-rare, max throughput
Semaphore         → resource pool / concurrency cap (N permits)
CountDownLatch    → wait for N events, single use
CyclicBarrier     → all-party sync point, reusable, phased algorithms
Phaser            → dynamic thread count, multi-phase
```

---

## 1. `volatile` — Visibility Only

```java
private volatile boolean shutdown = false;
```

**What it gives:**
- Visibility: write is flushed to main memory immediately; reads bypass CPU cache
- Memory barrier: no instruction reorder across the volatile access
- Happens-before: write HB subsequent read

**What it does NOT give:**
- Atomicity for compound operations

**Classic trap — this is a data race:**
```java
volatile int counter = 0;
counter++;  // BROKEN — three separate ops: read, increment, write
```

**When to use:**
- Single-writer / multiple-reader flag (`isShutdown`, `isReady`)
- Double-checked locking — the `instance` reference field must be volatile
- Status fields that only one thread writes

**When NOT to use:** Any read-modify-write operation with multiple writers.

---

## 2. `synchronized` — Mutual Exclusion + Visibility

```java
synchronized (lock) {
    // only one thread at a time
}
```

Every Java object has an intrinsic monitor. Acquiring it on contention causes an OS wait → context switch (10,000–100,000 ns cost).

**JVM optimizations under the hood:**
1. **Biased locking** (pre-JDK 15): single-thread fast path, zero CAS cost
2. **Thin lock / spin**: brief spin before blocking
3. **Inflated lock**: full OS mutex, context switch

**JDK 21+ note:** `synchronized` no longer pins virtual threads to carrier threads. Fully compatible with virtual threads — use freely.

**When to use:**
- Simple critical sections with low contention
- When you need `wait()` / `notify()` (Monitor pattern)
- When `ReentrantLock`'s extra features aren't needed

---

## 3. `volatile` vs `synchronized` vs `AtomicXxx` — Exact Difference

| Property | `volatile` | `synchronized` | `AtomicInteger` |
| --- | --- | --- | --- |
| Visibility | Yes | Yes | Yes |
| Mutual exclusion | No | Yes | No (lock-free) |
| Atomic compound ops | No | Yes | Yes (CAS-based) |
| Blocks threads | No | Yes | No |
| Performance under high contention | Fast (no lock) | Slow (OS wait) | Fast (CAS retry) |
| Performance under low contention | Fastest | Fast (biased lock) | Fast |

**Decision rule:**
- Read/write single primitive, single writer → `volatile`
- Read/write single primitive, multiple writers, atomic op needed → `AtomicXxx`
- Compound operation (check-then-act, read-modify-write on multiple fields) → `synchronized` or `ReentrantLock`

---

## 4. Atomic Classes — Lock-Free CAS

Package: `java.util.concurrent.atomic`

**Mechanism: Compare-And-Swap (CAS)**

CAS is a single CPU instruction (`CMPXCHG` on x86). No OS involvement. The CPU atomically checks if a memory location holds the expected value; if yes, replaces it; if no, returns false and the caller retries.

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();                      // atomic ++
counter.compareAndSet(5, 10);                   // set to 10 only if currently 5
int old = counter.getAndUpdate(x -> x * 2);    // functional atomic update
```

### AtomicReference

```java
AtomicReference<Config> config = new AtomicReference<>(initial);
Config current = config.get();
Config updated = new Config(current, newField);
config.compareAndSet(current, updated);  // atomic swap — safe under concurrency
```

**When:** Lock-free object publishing, snapshotting, immutable-object swap.

### ABA Problem

Value changes A → B → A. CAS wrongly succeeds because it only checks the value, not history. Fix: `AtomicStampedReference<T>` — CAS checks both value and integer version stamp.

### LongAdder vs AtomicLong

`LongAdder` splits the counter into a `base` field plus a `Cell[]` array. Under contention, each thread hashes to a different `Cell` and increments it independently. `sum()` adds all cells together.

| Scenario | `AtomicLong` | `LongAdder` |
| --- | --- | --- |
| Single thread | Same speed | Slightly slower |
| Low contention (< 4 threads) | Slightly faster | Equivalent |
| High contention (8+ threads) | Degrades (CAS spin loops) | Near-zero contention |
| Need exact live value | Yes — `get()` is exact | No — `sum()` is approximate during mutations |

**Rule:** Use `LongAdder` for metrics/stats/hit-counters. Use `AtomicLong` for sequence IDs or anything that needs an exact current value atomically.

---

## 5. `ReentrantLock` — Flexible Explicit Locking

```java
Lock lock = new ReentrantLock();
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();  // MUST be in finally — never skip
}
```

**Extra features over `synchronized`:**

| Feature | API |
| --- | --- |
| Non-blocking try | `lock.tryLock()` → returns false immediately if locked |
| Timed try | `lock.tryLock(1, TimeUnit.SECONDS)` |
| Interruptible lock | `lock.lockInterruptibly()` |
| Fairness (FIFO ordering) | `new ReentrantLock(true)` |
| Multiple conditions | `Condition c = lock.newCondition()` |
| Introspection | `lock.isLocked()`, `lock.getQueueLength()` |

### Multiple Conditions — Key Advantage

```java
// synchronized: one wait set per object — all waiters mixed together
// ReentrantLock: separate conditions, no spurious cross-wakeup

ReentrantLock lock = new ReentrantLock();
Condition notFull  = lock.newCondition();
Condition notEmpty = lock.newCondition();

// Producer waits on notFull, signals notEmpty
// Consumer waits on notEmpty, signals notFull
// No false wakeups of the wrong party
```

**When to use `ReentrantLock` over `synchronized`:**
- Need `tryLock()` — deadlock avoidance, timeout
- Need interruptible waiting
- Need fairness guarantee (prevent starvation)
- Need multiple distinct wait conditions
- Need lock introspection for monitoring/diagnostics

**JDK 21+ note:** Virtual threads don't pin on `ReentrantLock` — it releases the carrier thread during `lock()`. But `synchronized` is also fine in JDK 21+.

---

## 6. ReadWriteLock — Many Readers, Rare Writers

```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock  = rwLock.readLock();
Lock writeLock = rwLock.writeLock();

// Reader — many simultaneous readers allowed
readLock.lock();
try { return map.get(key); }
finally { readLock.unlock(); }

// Writer — exclusive, blocks all readers and writers
writeLock.lock();
try { map.put(key, value); }
finally { writeLock.unlock(); }
```

**Semantics:**
- Multiple readers can hold read lock simultaneously
- Write lock is exclusive — blocks until all current readers finish
- Writer acquisition blocks new readers from entering

**Hidden cost:** Read lock acquisition still performs a CAS on a shared counter → cache-line contention under many threads. Can be slower than a single `synchronized` at extreme read concurrency.

**When to use:** Config caches, DNS caches, routing tables — read 99% of the time, written rarely.

---

## 7. StampedLock — Optimistic Reads

Evolution of `ReadWriteLock`. Adds optimistic reads: read without acquiring any lock, then validate.

```java
StampedLock sl = new StampedLock();

// Optimistic read — zero lock cost
long stamp = sl.tryOptimisticRead();
int x = this.x;
int y = this.y;
if (!sl.validate(stamp)) {          // was it modified while we read?
    stamp = sl.readLock();          // fallback to real read lock
    try { x = this.x; y = this.y; }
    finally { sl.unlockRead(stamp); }
}
```

**Three modes:**

| Mode | Cost | When |
| --- | --- | --- |
| Write | Exclusive | Mutations |
| Read | Shared (CAS) | Normal reads |
| Optimistic read | Zero | Reads when writes are ultra-rare |

**Limitations:**
- NOT reentrant — same thread calling `readLock()` twice = deadlock
- Complex API — stamp must be tracked and passed back
- Don't mix virtual threads with `StampedLock` (pins carrier)

**When to use:** Spatial coordinates, immutable-ish objects, anything read thousands of times per write.

---

## 8. Semaphore — Resource Pool / Concurrency Cap

```java
Semaphore pool = new Semaphore(10);  // 10 permits = 10 concurrent users

pool.acquire();     // blocks if 0 permits available
try {
    // use resource (DB connection, HTTP slot, etc.)
} finally {
    pool.release(); // return permit
}
```

**NOT mutual exclusion.** Controls how many threads access something simultaneously.

**When to use:**
- Connection pools (limit concurrent DB connections)
- Rate limiting (max N concurrent in-flight requests)
- The parking lot problem — `new Semaphore(totalSpots)`
- Throttling access to expensive external services

**Binary semaphore (1 permit):** Equivalent to a mutex, but any thread can release it (unlike `synchronized` / `ReentrantLock` where only the acquiring thread releases). Useful for signaling between threads.

---

## 9. Coordination Primitives

### CountDownLatch — One-Shot Countdown

```java
CountDownLatch ready = new CountDownLatch(3);  // 3 workers must signal

// In each of 3 worker threads:
doSetup();
ready.countDown();              // decrement — doesn't block

// Main thread:
ready.await();                  // blocks until count reaches 0
startMainWork();
```

Count only goes down, never up. Cannot be reset — single use only.

**Use cases:**
- Wait for N services to initialize before starting server
- Simultaneous start: `new CountDownLatch(1)` as a starting gun — all workers call `await()`, main calls `countDown()`
- Fan-out/fan-in: launch N parallel tasks, `await()` until all finish

### CyclicBarrier — Reusable All-Party Sync

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    mergePhaseResult();  // runs when all 3 arrive, before any continue
});

// Each of 3 worker threads calls per phase:
doPhaseWork();
barrier.await();  // wait for all 3 — then all continue simultaneously
doNextPhase();
barrier.await();  // reuse for next phase
```

| | `CountDownLatch` | `CyclicBarrier` |
| --- | --- | --- |
| Reusable | No | Yes — auto-resets |
| Who waits | One or many callers | All participants |
| Count tracks | Events / tasks | Threads |
| Completion action | None | Optional `Runnable` |

**When to use:** Iterative parallel algorithms (parallel sort phases, simulation steps), multi-stage pipelines.

### Phaser — Dynamic Flexible Barrier

Most powerful, most complex. Replaces both `CountDownLatch` and `CyclicBarrier` when you need threads to join or leave mid-execution.

```java
Phaser phaser = new Phaser(1);  // 1 = main thread registers itself

// Threads join dynamically
phaser.register();
phaser.arriveAndAwaitAdvance();  // sync point
phaser.arriveAndDeregister();    // thread done, leaves

// Main thread releases
phaser.arriveAndDeregister();
```

**Override `onAdvance()`** to control when the phaser terminates.

**When to use:** Recursive decomposition, workflows where thread count varies per phase, anything where neither `CountDownLatch` nor `CyclicBarrier` is flexible enough.

---

## 10. Thread-Safe Collections

### ConcurrentHashMap

**Mechanism (Java 8+):** Lock-per-bucket (lock striping). Each bucket has its own lock. CAS for empty-slot insertions. Reads are fully lock-free.

```java
ConcurrentHashMap<K, V> map = new ConcurrentHashMap<>();
map.putIfAbsent(key, value);                            // atomic
map.compute(key, (k, v) -> v == null ? 1 : v + 1);     // atomic read-modify-write
map.computeIfAbsent(key, k -> new ArrayList<>());       // atomic get-or-create
```

**vs `Collections.synchronizedMap()`:** That wraps every call in `synchronized(this)` — single global lock, total serialization. `ConcurrentHashMap` allows N concurrent writes to N different buckets.

**vs `Hashtable`:** Same as synchronizedMap — legacy, avoid.

### CopyOnWriteArrayList

Every mutation copies the entire backing array, writes to the copy, then atomically replaces the reference. Readers see a snapshot and are never blocked.

```java
CopyOnWriteArrayList<Listener> listeners = new CopyOnWriteArrayList<>();
listeners.add(l);       // O(N) — copies entire array
for (Listener l : listeners) { l.onEvent(e); }  // O(1), zero lock
```

O(N) writes. O(1) reads with zero contention. Use for listener lists, event handler registrations — written rarely, iterated very frequently. Wrong choice for large frequently-mutated collections.

### LinkedBlockingQueue vs ArrayBlockingQueue

Both implement `BlockingQueue`: `put()` blocks when full, `take()` blocks when empty.

| | `LinkedBlockingQueue` | `ArrayBlockingQueue` |
| --- | --- | --- |
| Capacity | Optionally bounded (default: `Integer.MAX_VALUE`) | Always bounded |
| Backing structure | Linked nodes (heap allocated) | Circular array |
| Locks | Two: separate head and tail lock | One: shared lock |
| Producer-consumer contention | Lower (separate locks) | Higher (shared lock) |
| Fairness option | No | Yes |

### Other Key Collections

| Collection | Mechanism | When |
| --- | --- | --- |
| `ConcurrentLinkedQueue` | Lock-free CAS | High-throughput non-blocking FIFO |
| `ConcurrentSkipListMap` | Lock-free skip list | Sorted concurrent map |
| `PriorityBlockingQueue` | Heap + single lock | Priority-ordered blocking queue |
| `SynchronousQueue` | No buffer — direct handoff | Task handoff / thread rendezvous |
| `LinkedTransferQueue` | SynchronousQueue + optional buffer | Handoff with optional buffering |

---

## 11. Thread Contention — What It Is, How to Measure It

**Contention:** Multiple threads competing for the same lock or atomic variable. Waiting threads are descheduled by the OS. Cost per context switch: 10,000–100,000 ns.

**Symptoms:**
- Thread count rises, throughput doesn't improve or degrades
- High CPU, low actual work rate
- Thread dump shows many `BLOCKED` threads on the same monitor address

### How to Measure

**Thread dump:**
```bash
jstack <pid>   # look for BLOCKED threads with same "waiting to lock <0x...>"
```

**JFR (Java Flight Recorder) — best tool:**
```bash
jcmd <pid> JFR.start duration=60s filename=recording.jfr
# Open in JMC → "Lock Instances" view shows hot monitors, wait times
```

**JMX / ThreadMXBean:**
```java
ThreadMXBean mbean = ManagementFactory.getThreadMXBean();
mbean.setThreadContentionMonitoringEnabled(true);
ThreadInfo info = mbean.getThreadInfo(id, true, true);
info.getBlockedCount();   // times this thread blocked for a monitor
info.getBlockedTime();    // ms spent blocked
```

### Contention Behavior Per Primitive

| Primitive | Under Contention |
| --- | --- |
| `synchronized` | OS wait → context switch |
| `ReentrantLock` | Same — but `tryLock()` lets you bail out immediately |
| `AtomicXxx` (CAS) | CPU spin and retry — burns CPU but no context switch |
| `LongAdder` | Near-zero — splits per cell, threads don't compete |
| `ConcurrentHashMap` | Contention only within same bucket |
| `volatile` | No contention possible — no lock |
| `StampedLock` optimistic | Zero until validation fails, then falls back |

---

## 12. Lock Striping — Fine-Grained Locking

**Problem:** One global lock serializes all threads even when they access different data.

**Solution:** Partition data, give each partition its own lock. Only threads accessing the same partition contend.

```java
// Naive — all threads compete for one lock
synchronized (globalLock) { map.put(k, v); }

// Striped — only threads in same bucket compete
private static final int STRIPES = 16;
private final Object[] locks = new Object[STRIPES];

void put(K key, V value) {
    int stripe = Math.abs(key.hashCode() % STRIPES);
    synchronized (locks[stripe]) {
        segments[stripe].put(key, value);
    }
}
```

`ConcurrentHashMap` does exactly this internally: CAS for empty buckets (zero lock), `synchronized` on bucket head node for non-empty buckets.

**Guava `Striped<Lock>`** gives a ready-made utility:
```java
Striped<Lock> striped = Striped.lock(64);
Lock lock = striped.get(key);
lock.lock(); try { ... } finally { lock.unlock(); }
```

**Rules for correct striping:**
1. Stripe by a stable natural key (hash, ID, shard number)
2. Stripe count should be >= expected concurrent thread count
3. Always acquire multiple locks in a consistent total order to prevent deadlock
4. More stripes = less contention = more memory overhead

---

## 13. Monitor Pattern

Classic Java pattern using `synchronized` + `wait()` / `notify()`.

```java
class BoundedBuffer<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    synchronized void put(T item) throws InterruptedException {
        while (queue.size() == capacity) wait(); // ALWAYS while — not if
        queue.add(item);
        notifyAll(); // wake all waiters
    }

    synchronized T take() throws InterruptedException {
        while (queue.isEmpty()) wait();
        T item = queue.remove();
        notifyAll();
        return item;
    }
}
```

**Critical rules:**
- `wait()` must be called inside `synchronized` on the same object
- Always use `while` loop — not `if`. Spurious wakeups are real; re-check the condition after every wake
- `notifyAll()` is safer than `notify()` — `notify()` wakes one arbitrary waiter, which may not be the right one
- The downside of `notifyAll()`: wakes all waiters, only one proceeds, others immediately go back to sleep — wasteful under high contention

**Better with `ReentrantLock`:** Two separate `Condition` objects eliminate spurious cross-wakeups:
```java
ReentrantLock lock = new ReentrantLock();
Condition notFull  = lock.newCondition();
Condition notEmpty = lock.newCondition();
// Producer signals only notEmpty; consumer signals only notFull
```

---

## 14. Producer-Consumer Pattern

### Canonical Java — Use `BlockingQueue`

```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(100);  // bounded = backpressure

// Producer thread
ExecutorService producer = Executors.newSingleThreadExecutor();
producer.submit(() -> {
    while (running) {
        Task t = generateTask();
        queue.put(t);  // blocks if full — natural backpressure to producer
    }
});

// Consumer threads
ExecutorService consumers = Executors.newFixedThreadPool(4);
for (int i = 0; i < 4; i++) {
    consumers.submit(() -> {
        while (running) {
            Task t = queue.take();  // blocks if empty
            process(t);
        }
    });
}
```

`BlockingQueue` handles all wait/notify internally. Never write manual `wait()`/`notify()` for producer-consumer — use `BlockingQueue`.

**Bounded vs unbounded:**
- Bounded (`ArrayBlockingQueue`, `LinkedBlockingQueue(N)`): backpressure — producer slows when consumer is slow. Prevents memory exhaustion.
- Unbounded (`LinkedBlockingQueue()`): producer never blocks — queue grows unbounded if consumer is slow. Only safe when producer is naturally rate-limited.

---

## 15. Reader-Writer Pattern

```java
class ConfigCache {
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private Map<String, String> config = new HashMap<>();

    String get(String key) {
        rwLock.readLock().lock();
        try { return config.get(key); }
        finally { rwLock.readLock().unlock(); }
    }

    void reload(Map<String, String> newConfig) {
        rwLock.writeLock().lock();
        try { config = newConfig; }
        finally { rwLock.writeLock().unlock(); }
    }
}
```

**With `StampedLock` for maximum read performance:**
```java
StampedLock sl = new StampedLock();

String get(String key) {
    long stamp = sl.tryOptimisticRead();
    String val = config.get(key);
    if (!sl.validate(stamp)) {         // writer modified during our read
        stamp = sl.readLock();
        try { val = config.get(key); }
        finally { sl.unlockRead(stamp); }
    }
    return val;
}
```

---

## 16. Interview Application: Parking Lot Example

**What "2/3 concurrency" sounds like:**
> "The collection needs to be thread-safe. I'll use per-spot locking."

**What "3/3 concurrency" sounds like:**
> "Each `ParkingSpot` holds its own `ReentrantLock`. On assignment, the entrance gate calls `tryLock()` — if another gate is mid-assignment on that spot, we skip it immediately and try the next one. No blocking. `ParkingCollection` uses two `ConcurrentHashMap`s — one for available spots, one for occupied. `ConcurrentHashMap` gives lock-free reads and bucket-level write locks. This means 100 gates can operate in parallel across 100 different spots with zero global lock."

```java
class ParkingSpot {
    private final ReentrantLock lock = new ReentrantLock();
    private boolean available = true;

    boolean tryAssign(Vehicle v) {
        if (lock.tryLock()) {
            try {
                if (available) {
                    available = false;
                    return true;
                }
            } finally { lock.unlock(); }
        }
        return false;  // another gate is assigning this spot right now — skip
    }

    void release() {
        lock.lock();
        try { available = true; }
        finally { lock.unlock(); }
    }
}

class ParkingCollection {
    private final ConcurrentHashMap<String, ParkingSpot> available = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, ParkingSpot> occupied  = new ConcurrentHashMap<>();

    Optional<ParkingSpot> assign(Vehicle vehicle) {
        for (ParkingSpot spot : available.values()) {
            if (spot.tryAssign(vehicle)) {
                available.remove(spot.getId());
                occupied.put(vehicle.getId(), spot);
                return Optional.of(spot);
            }
        }
        return Optional.empty();  // lot full
    }

    void release(String vehicleId) {
        ParkingSpot spot = occupied.remove(vehicleId);
        if (spot != null) {
            spot.release();
            available.put(spot.getId(), spot);
        }
    }
}
```

---

## 17. Full Decision Tree

```
Need thread safety?
│
├─ Single primitive field, single writer
│   → volatile
│
├─ Single primitive, multiple writers, atomic op
│   → AtomicInteger / AtomicLong
│
├─ Counter, high contention (stats, metrics)
│   → LongAdder
│
├─ Lock-free object reference swap / publish
│   → AtomicReference
│
├─ Simple critical section, low contention
│   → synchronized
│
├─ Need tryLock / timeout / interruptible / multiple conditions
│   → ReentrantLock
│
├─ Many readers, rare writers
│   → ReentrantReadWriteLock
│
├─ Reads dominate, writes ultra-rare, max performance
│   → StampedLock (optimistic reads)
│
├─ Resource pool / concurrency cap
│   → Semaphore
│
├─ Wait for N events, one-time
│   → CountDownLatch
│
├─ All threads sync at repeated barriers
│   → CyclicBarrier
│
├─ Dynamic thread participation, multi-phase
│   → Phaser
│
├─ Thread-safe key-value store
│   → ConcurrentHashMap
│
├─ Read-heavy list, rare writes
│   → CopyOnWriteArrayList
│
└─ Producer-consumer queue
    → LinkedBlockingQueue (optionally bounded)
    → ArrayBlockingQueue (always bounded, fairness option)
```
