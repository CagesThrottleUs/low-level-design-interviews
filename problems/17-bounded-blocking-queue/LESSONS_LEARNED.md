# Lessons Learned — Bounded Blocking Queue

---

## Mistake 1: `if` Instead of `while` — Spurious Wakeup Bug

**What happens:**
```java
if (queue.size() == capacity) { notFull.await(); }
queue.addLast(item); // runs even on spurious wakeup when queue is still full
```

**Why it's wrong:** `Condition.await()` can return spuriously — the OS wakes the thread for reasons unrelated to a `signal()`. This is allowed by the JVM spec and happens on Linux (pthreads). With `if`, the thread proceeds without re-checking the condition and adds an item to a full queue.

**Fix:** `while` loop — re-checks condition after every wakeup:
```java
while (queue.size() == capacity) { notFull.await(); }
// Spurious wakeup: loops back, re-checks, goes back to sleep
```

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 2: `lock.unlock()` Not in `finally`

**What happens:**
```java
lock.lock();
queue.addLast(item); // throws OutOfMemoryError
notEmpty.signal();
lock.unlock();       // never reached — lock held forever
```

**Why it's wrong:** Any exception between `lock.lock()` and `lock.unlock()` leaves the lock permanently held. All other threads waiting on the lock deadlock.

**Fix:** Always unlock in `finally`:
```java
lock.lock();
try {
    // operations
} finally {
    lock.unlock(); // executes even if exception thrown
}
```

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 3: Signaling the Wrong Condition

**What happens:**
```java
public void put(T item) throws InterruptedException {
    lock.lock();
    try {
        while (queue.size() == capacity) notFull.await();
        queue.addLast(item);
        notFull.signal(); // WRONG — should signal notEmpty
    } finally { lock.unlock(); }
}
```

**Why it's wrong:** After adding an item, we should wake a sleeping CONSUMER (`notEmpty.signal()`). Signaling `notFull` wakes a sleeping producer — who checks the queue, finds it not full, adds another item. Consumers never wake up. Deadlock when queue fills.

**Fix:** Cross-reference before committing code:
- `put()` signals `notEmpty` (wake consumers)
- `take()` signals `notFull` (wake producers)

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 4: One Condition Variable — `notifyAll()` Thundering Herd

**What happens:** Using `synchronized` + one `wait()`/`notifyAll()`:
```java
synchronized void put(T item) throws InterruptedException {
    while (queue.size() == capacity) wait();
    queue.addLast(item);
    notifyAll(); // wakes ALL threads — producers and consumers
}
```

**Why it's a gap:** `notifyAll()` wakes every sleeping thread. If 50 producers and 50 consumers are waiting: adding one item wakes all 100. Only one consumer can actually take the item; others check conditions and go back to sleep. Wasteful at high throughput.

**Fix:** Two `Condition` objects:
- `notEmpty.signal()` in `put()` — wakes exactly ONE consumer
- `notFull.signal()` in `take()` — wakes exactly ONE producer

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 5: Using JDK BlockingQueue

**What happens:** `private final BlockingQueue<T> queue = new LinkedBlockingQueue<>(capacity)`.

**Why it's wrong:** The interview problem explicitly tests whether you can BUILD the primitive. Using `LinkedBlockingQueue` is like answering "implement a linked list" by using `java.util.LinkedList`. Interviewer will ask you to implement from scratch.

**Correct stance:** "I know `LinkedBlockingQueue` exists and is the production choice. The interview question asks me to build the underlying mechanism using `ReentrantLock` and `Condition`."

**Gap dimension:** Requirements / Communication (Dimensions 1, 7)

---

## Self-Assessment Checklist

- [ ] Is the condition check `while` (not `if`)?
- [ ] Is `lock.unlock()` in a `finally` block?
- [ ] Are there TWO condition variables (`notFull` and `notEmpty`)?
- [ ] Does `put()` signal `notEmpty` (not `notFull`)?
- [ ] Does `take()` signal `notFull` (not `notEmpty`)?
- [ ] Is `signal()` used (not `signalAll()`) with two conditions?
- [ ] Is `ArrayDeque` used (not `LinkedList`)?
- [ ] Is JDK's `BlockingQueue` NOT used?

---

## Two-Condition Cheat Sheet

```
put():
  lock()
  while full → await(notFull)   ← producers sleep here
  addLast(item)
  signal(notEmpty)               ← wake ONE consumer
  unlock()

take():
  lock()
  while empty → await(notEmpty)  ← consumers sleep here
  removeFirst()
  signal(notFull)                ← wake ONE producer
  unlock()
```

---

## Remediation Targets

| Dimension | If scored 0–1 | Go to |
|-----------|--------------|-------|
| Concurrency | if not while, wrong signal direction, no finally | GAP_REMEDIATION.md#gap-6 — entire section |
| Communication | Used JDK BlockingQueue without explaining why not | GAP_REMEDIATION.md#gap-7 |
