# Solution Guide — Bounded Blocking Queue

**Read after your attempt.**

---

## The Core Design Decision: Two Conditions

`synchronized` + `wait/notify` gives you ONE wait-set per object. `notify()` wakes one random thread — could be a producer when you want a consumer, or vice versa. This forces `notifyAll()` which is wasteful.

`ReentrantLock` + two `Condition` objects gives you TWO separate wait-sets:
- Producers sleep in `notFull` — woken ONLY by consumers
- Consumers sleep in `notEmpty` — woken ONLY by producers

Targeted wakeups. `signal()` (not `signalAll()`) sufficient. Zero wasted wakeups.

---

## Full Implementation

```java
import java.util.ArrayDeque;
import java.util.Deque;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class BoundedBlockingQueue<T> {

    private final Deque<T>  queue;
    private final int       capacity;
    private final ReentrantLock lock;
    private final Condition notFull;   // producers wait here
    private final Condition notEmpty;  // consumers wait here

    public BoundedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException("Capacity must be > 0");
        this.capacity = capacity;
        this.queue    = new ArrayDeque<>(capacity);
        this.lock     = new ReentrantLock();
        this.notFull  = lock.newCondition();
        this.notEmpty = lock.newCondition();
    }

    /**
     * Blocks until space is available, then inserts.
     * Propagates InterruptedException — caller decides policy.
     */
    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            // WHILE not IF — spurious wakeup guard
            while (queue.size() == capacity) {
                notFull.await();             // release lock + sleep
            }
            queue.addLast(item);
            notEmpty.signal();               // wake ONE waiting consumer
        } finally {
            lock.unlock();                   // ALWAYS in finally
        }
    }

    /**
     * Blocks until an item is available, then removes and returns it.
     */
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();            // release lock + sleep
            }
            T item = queue.removeFirst();
            notFull.signal();               // wake ONE waiting producer
            return item;
        } finally {
            lock.unlock();
        }
    }

    /** Non-blocking insert. Returns false immediately if full. */
    public boolean offer(T item) {
        lock.lock();
        try {
            if (queue.size() == capacity) return false;
            queue.addLast(item);
            notEmpty.signal();
            return true;
        } finally {
            lock.unlock();
        }
    }

    /** Non-blocking remove. Returns null immediately if empty. */
    public T poll() {
        lock.lock();
        try {
            if (queue.isEmpty()) return null;
            T item = queue.removeFirst();
            notFull.signal();
            return item;
        } finally {
            lock.unlock();
        }
    }

    /** Timed blocking offer. Returns false if timeout expires. */
    public boolean offer(T item, long timeout, TimeUnit unit) throws InterruptedException {
        long nanosRemaining = unit.toNanos(timeout);
        lock.lock();
        try {
            while (queue.size() == capacity) {
                if (nanosRemaining <= 0) return false;
                nanosRemaining = notFull.awaitNanos(nanosRemaining);
            }
            queue.addLast(item);
            notEmpty.signal();
            return true;
        } finally {
            lock.unlock();
        }
    }

    public int size() {
        lock.lock();
        try { return queue.size(); }
        finally { lock.unlock(); }
    }

    public boolean isEmpty() {
        lock.lock();
        try { return queue.isEmpty(); }
        finally { lock.unlock(); }
    }

    public boolean isFull() {
        lock.lock();
        try { return queue.size() == capacity; }
        finally { lock.unlock(); }
    }
}
```

---

## The Four Non-Negotiable Rules

**Rule 1: `while` not `if` on condition check**

`Condition.await()` can return spuriously (OS-level interrupt with no signal). After waking, the condition might still not hold. `while` loops back and re-checks. `if` proceeds on a spurious wakeup → corrupted state.

```java
// WRONG
if (queue.size() == capacity) { notFull.await(); }
// Spurious wakeup: adds to full queue — corruption

// CORRECT
while (queue.size() == capacity) { notFull.await(); }
// Always re-checks after wakeup
```

**Rule 2: `lock.unlock()` in `finally`**

If `queue.addLast(item)` throws (memory error), the lock must still be released. Without `finally`, the lock is permanently held → deadlock for all other threads.

**Rule 3: Signal the CORRECT Condition**

`put()` signals `notEmpty` (wakes consumers). `take()` signals `notFull` (wakes producers). Swapping them (signaling your own condition type) wakes nobody useful — silent deadlock.

**Rule 4: Use `signal()` not `signalAll()` with two separate Conditions**

Each condition has only one type of waiter. When a consumer takes an item, exactly one producer can proceed — wake exactly one. `signalAll()` wakes all producers to compete for one slot — thundering herd.

---

## Why `ArrayDeque` Not `LinkedList`

- `ArrayDeque` stores elements in a contiguous array — cache-friendly, no per-node allocation
- `LinkedList` allocates a `Node` object per element — GC pressure under high throughput
- Both are O(1) for addLast/removeFirst
- For high-throughput queues, `ArrayDeque` wins on GC pauses

---

## Demonstration (Multi-threaded)

```java
public class Demo {
    public static void main(String[] args) throws InterruptedException {
        BoundedBlockingQueue<Integer> q = new BoundedBlockingQueue<>(5);

        // 3 producers
        for (int p = 0; p < 3; p++) {
            final int pid = p;
            new Thread(() -> {
                for (int i = 0; i < 10; i++) {
                    try {
                        q.put(pid * 10 + i);
                        System.out.printf("P%d put %d  size=%d%n", pid, pid * 10 + i, q.size());
                        Thread.sleep(50);
                    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                }
            }, "Producer-" + p).start();
        }

        // 2 consumers
        for (int c = 0; c < 2; c++) {
            final int cid = c;
            new Thread(() -> {
                for (int i = 0; i < 15; i++) {
                    try {
                        int val = q.take();
                        System.out.printf("C%d took %d  size=%d%n", cid, val, q.size());
                        Thread.sleep(80);
                    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                }
            }, "Consumer-" + c).start();
        }
    }
}
```

---

## Extension: Fair Queue

```java
// FIFO thread ordering — threads wake in the order they blocked
this.lock = new ReentrantLock(true); // fair = true
// Trade-off: lower throughput (~30% slower) due to OS scheduling overhead
// Use only when starvation is a real concern
```

---

## What Strong Candidates Do Differently

- State the two-Condition design upfront before writing code — explains WHY
- `while` loop for spurious wakeup: explains the JVM spec reason
- `finally` block: explains the lock-leak scenario without it
- Signal the correct condition — explicitly cross-checks "put signals notEmpty, take signals notFull"
- `ArrayDeque` over `LinkedList` — explains GC reason
- Mention `offer(timeout)` as a production-ready variant

## What Average Candidates Miss

- `if` instead of `while` — spurious wakeup silent bug
- `lock.unlock()` not in `finally` — lock leak on exception
- Signaling the wrong condition (notFull after put instead of notEmpty)
- `signalAll()` everywhere — unnecessary wakeups
- Using `synchronized` + `wait/notify` with ONE condition → can't separate producer/consumer wait-sets
