# C++ Concurrency — Complete Reference (C++20/23/26)

---

## Foundation: The C++ Memory Model

C++ gives you explicit control over three properties that Java's model hides behind a single `volatile` keyword:

| Property | Meaning |
| --- | --- |
| **Visibility** | Write by one thread seen by others |
| **Atomicity** | Operation completes without partial intermediate state |
| **Ordering** | Instructions execute in the sequence the code implies (CPUs and compilers reorder for speed) |

C++ lets you choose exactly how much ordering guarantee you pay for at each atomic operation. Java `volatile` always gives you the full `seq_cst` guarantee — C++ lets you dial it down.

**Key C++ advantage:** No object has a hidden monitor or reference-count header. You only pay for what you explicitly place. A plain `int` is zero overhead.

---

## The Mutex Family

### `std::mutex` — Use This 95% of the Time

```cpp
#include <mutex>

std::mutex mtx;
{
    std::lock_guard<std::mutex> lk(mtx);  // locks on construction
    shared_data.push_back(42);
}  // lk destructor unlocks — always, even on exception
```

NOT reentrant. Same thread calling `lock()` twice = deadlock. No hidden overhead on objects that don't use it.

### `std::recursive_mutex`

Same thread can acquire N times without deadlock. Must release N times. Slower than `mutex` due to ownership tracking overhead.

**When to use:** Class methods that hold the mutex call other class methods that also need it. This is usually a design smell — prefer restructuring the class so inner methods don't lock. Use `recursive_mutex` only when refactoring is too costly.

```cpp
std::recursive_mutex rmtx;

void outer() {
    std::lock_guard<std::recursive_mutex> lk(rmtx);
    inner();  // re-acquires rmtx — fine with recursive_mutex
}

void inner() {
    std::lock_guard<std::recursive_mutex> lk(rmtx);  // re-entrant
    // ...
}
```

### `std::shared_mutex` (C++17) — Readers-Writer Lock

Multiple threads hold shared (read) locks simultaneously. Write lock is exclusive — blocks all readers and writers.

```cpp
#include <shared_mutex>

std::shared_mutex rw_mtx;

// Reader — many simultaneous readers allowed
std::shared_lock<std::shared_mutex> rl(rw_mtx);
auto val = map.find(key);

// Writer — exclusive, blocks until all readers finish
std::unique_lock<std::shared_mutex> wl(rw_mtx);
map[key] = value;
```

**When to use:** Read-heavy data — caches, config, routing tables. Rule of thumb: reads outnumber writes 10:1 or more.

### `std::timed_mutex` / `std::shared_timed_mutex`

Add `try_lock_for(duration)` and `try_lock_until(time_point)`. Use for deadlock avoidance via timeouts.

---

## RAII Lock Wrappers

RAII = Resource Acquisition Is Initialization. Lock acquired on construction, released on destruction — always, even on exception.

### `std::lock_guard<M>` — Simplest

```cpp
std::lock_guard<std::mutex> lk(mtx);  // locks here
// critical section
// destructor unlocks here — no manual management needed
```

Zero overhead beyond the mutex itself. No manual unlock. No deferred lock. No try-lock. If you just need a scope-bounded lock, use this.

### `std::unique_lock<M>` — Full-Featured

```cpp
// Deferred lock — acquire later
std::unique_lock<std::mutex> lk(mtx, std::defer_lock);
do_work_without_lock();
lk.lock();

// Condition variable — required by condition_variable::wait()
cv.wait(lk, [] { return ready; });  // releases mutex during wait

// Try-lock
if (lk.try_lock()) { ... }

// Manual unlock mid-scope
lk.unlock();
do_expensive_work_outside_lock();
lk.lock();
```

Required whenever you use `condition_variable`. Movable — can transfer lock ownership.

### `std::shared_lock<M>` (C++14) — Shared Read Lock

```cpp
std::shared_lock<std::shared_mutex> rl(rw_mtx);
// multiple threads can hold this simultaneously
```

### `std::scoped_lock` (C++17) — Multiple Mutexes, No Deadlock

```cpp
std::scoped_lock lk(m1, m2);  // locks both atomically — deadlock-free
// same as calling std::lock(m1, m2) but RAII
```

**Decision table:**

| Scenario | Use |
| --- | --- |
| Simple critical section | `lock_guard` |
| Condition variable | `unique_lock` |
| Need try-lock / timed-lock | `unique_lock` |
| Lock two+ mutexes atomically | `scoped_lock(m1, m2)` |
| Shared read on `shared_mutex` | `shared_lock` |
| Exclusive write on `shared_mutex` | `unique_lock` |

---

## `std::condition_variable` — Efficient Blocking

Blocks a thread until a condition becomes true. Avoids busy-polling. Always paired with `unique_lock<mutex>`.

```cpp
std::mutex mtx;
std::condition_variable cv;
std::queue<int> buffer;
const size_t MAX_SIZE = 10;
bool done = false;

// Producer thread
void producer() {
    for (int i = 0; i < 100; ++i) {
        std::unique_lock<std::mutex> lk(mtx);
        cv.wait(lk, [&]{ return buffer.size() < MAX_SIZE; }); // wait if full
        buffer.push(i);
        lk.unlock();
        cv.notify_one();
    }
    {
        std::lock_guard<std::mutex> lk(mtx);
        done = true;
    }
    cv.notify_all();
}

// Consumer thread
void consumer() {
    while (true) {
        std::unique_lock<std::mutex> lk(mtx);
        cv.wait(lk, [&]{ return !buffer.empty() || done; });
        if (buffer.empty()) break;
        int val = buffer.front();
        buffer.pop();
        lk.unlock();
        cv.notify_one();
        process(val);
    }
}
```

**Critical rules:**
- Always use the predicate form `wait(lk, pred)`. The kernel can wake a thread spuriously (no notification). The predicate loops until the condition is actually true.
- `notify_all()` when multiple threads may wait on the same condition; `notify_one()` when there is one consumer.
- `condition_variable` works only with `unique_lock<mutex>`. For other lockables, use `condition_variable_any` (slower).

---

## `std::atomic<T>` — Lock-Free Operations

No mutex. CPU-level atomic instruction (`LOCK XCHG`, `CMPXCHG` on x86). No OS involvement.

```cpp
#include <atomic>

std::atomic<int> counter{0};
counter.fetch_add(1, std::memory_order_relaxed);  // atomic increment
counter.store(0);
int val = counter.load();
bool ok = counter.compare_exchange_strong(expected, desired);  // CAS
```

**Does NOT replace mutex for compound operations spanning two variables.** Atomics are per-variable. Multi-variable atomicity requires a mutex.

### `std::atomic_ref<T>` (C++20) — Atomic on Non-Atomic Object

Applies atomic operations to a plain, non-atomic object. Useful for:
- Legacy code where you can't change the field type to `atomic<T>`
- C interop where struct layout must stay as plain types
- Objects that are atomic in some phases, non-atomic in others

```cpp
int plain_int = 0;  // can't change this — C API constraint
std::atomic_ref<int> ref(plain_int);
ref.fetch_add(1);   // atomic
// plain_int is now 1
```

**Constraint:** ALL concurrent accesses to the object must go through `atomic_ref` instances. Mixed atomic/non-atomic access = undefined behavior.

---

## Memory Orders — Fine-Grained Control

The memory order argument on every atomic operation controls how that operation is ordered relative to all other memory accesses. Default is `seq_cst` (safest, slowest).

### `memory_order_relaxed` — Atomicity Only

No ordering constraint. Only guarantees the operation itself is atomic.

```cpp
std::atomic<int> hits{0};
hits.fetch_add(1, std::memory_order_relaxed);  // fast stats counter
```

No visibility guarantee for other variables. Thread A incrementing `hits` with `relaxed` gives Thread B no guarantee about what other writes Thread A did before the increment. Use ONLY for truly independent counters.

### `memory_order_release` (stores) + `memory_order_acquire` (loads)

The canonical publish/subscribe pattern. "I wrote data; you may read it now."

```cpp
int data = 0;
std::atomic<bool> ready{false};

// Thread A — producer
data = 42;                                        // (1) write data first
ready.store(true, std::memory_order_release);     // (2) publish

// Thread B — consumer
while (!ready.load(std::memory_order_acquire));   // (3) wait for publish
assert(data == 42);                               // (4) guaranteed visible
```

The `release` at (2) synchronizes-with the `acquire` at (3). Everything Thread A wrote before (2) is guaranteed visible to Thread B after (3).

**Mental model:** `release` = "I'm done writing; everything before this store is committed." `acquire` = "I'm about to read; give me everything that was committed before this load."

### `memory_order_acq_rel` — Both Acquire and Release

For read-modify-write operations (fetch_add, exchange, compare_exchange) that both read prior state and publish new state.

```cpp
std::atomic<int> x;
x.fetch_add(1, std::memory_order_acq_rel);
```

### `memory_order_seq_cst` — Total Global Order (Default)

All `seq_cst` operations across all threads appear in one total consistent order, as seen by every thread. Most expensive — requires a full memory fence (`MFENCE` or `LOCK` prefix on x86). Only needed when multiple atomic variables must be seen in the same order by all threads simultaneously.

**When in doubt, use `seq_cst`.** It is always correct. Only optimize to weaker orders when you have profiled a hot path and understand the causality requirements exactly.

### Memory Order Decision Guide

| Need | Order |
| --- | --- |
| Just atomicity, no cross-thread visibility | `relaxed` |
| Publish data from producer to consumer (store) | `release` |
| Consume published data (load) | `acquire` |
| RMW that both reads and publishes | `acq_rel` |
| Multiple atomics, global consistent ordering | `seq_cst` |
| Unsure | `seq_cst` |

**vs Java:** Java `volatile` = always `seq_cst`. Java has no weaker option. C++ lets you pay only for the ordering you actually need.

---

## C++20 High-Level Primitives

### `std::counting_semaphore<N>` / `std::binary_semaphore`

Controls concurrent access to N resources. `acquire()` decrements (blocks at 0). `release()` increments. Unlike a mutex — ANY thread can release, not just the acquirer.

```cpp
#include <semaphore>

std::counting_semaphore<3> pool{3};  // 3 database connections

void use_connection() {
    pool.acquire();   // blocks if 0 permits
    // ... use connection ...
    pool.release();   // any thread can call this
}

// Binary semaphore — lightweight thread signaling
std::binary_semaphore sig{0};
// Thread A: sig.release();  // signal
// Thread B: sig.acquire();  // wait
```

**vs `condition_variable`:** Semaphore has no spurious wakeups and requires no mutex. Better for simple resource counting or one-shot signaling. Use `condition_variable` when the wait condition is complex or composite.

### `std::latch` (C++20) — One-Shot Countdown

```cpp
#include <latch>

std::latch done{5};  // count starts at 5

// In 5 worker threads — decrement and CONTINUE (don't block)
do_work();
done.count_down();

// Main thread — blocks until count reaches 0
done.wait();
```

One-time use. Cannot reset. Workers don't wait — they continue after `count_down()`. Only `wait()` callers block.

**Use for:** Fan-out/fan-in — launch N tasks, wait for all. Simpler than futures for fire-and-join.

### `std::barrier<F>` (C++20) — Reusable Phase Sync

```cpp
#include <barrier>

auto on_completion = []() noexcept {
    merge_phase_result();  // runs when all threads arrive, before any continue
};
std::barrier bar{N, on_completion};

// In each of N threads, per phase:
do_phase_work();
bar.arrive_and_wait();  // all N must arrive — completion fn runs — barrier resets
do_next_phase();
bar.arrive_and_wait();  // reuse for next phase

// Thread can leave early
bar.arrive_and_drop();  // arrive for this phase, deregister permanently
```

**vs `latch`:** Reusable — auto-resets after all threads arrive. Has a completion function that runs between phases. Threads can drop out dynamically.

**When to use:** Iterative parallel algorithms (simulation steps, parallel sort phases, multi-pass rendering), any phased parallel work.

### `std::jthread` + `std::stop_token` (C++20) — Structured, Cancellable Threads

`std::jthread` joins automatically on destruction. Supports cooperative cancellation via `stop_token`.

```cpp
#include <thread>

std::jthread worker([](std::stop_token st) {
    while (!st.stop_requested()) {   // cooperative check at safe points
        do_chunk_of_work();
    }
    cleanup();
});  // destructor calls request_stop() then joins — no dangling threads

// From another thread:
worker.request_stop();  // sets stop_requested() = true; thread exits at next check
```

**vs Java `Thread.interrupt()`:** Java throws `InterruptedException` — invasive, forces try-catch everywhere, can unwind across call stacks unexpectedly. C++ `stop_token` is poll-based — the thread checks at safe points it controls. No unwinding surprises.

**Auto-join on destruction:** `std::thread` terminates the process if destroyed while joinable. `std::jthread` joins — no memory leaks, no UB.

### `std::atomic<shared_ptr<T>>` (C++20) — Lock-Free Config Reload

```cpp
std::atomic<std::shared_ptr<Config>> config;

// Writer — atomic swap of entire config object
config.store(std::make_shared<Config>(new_cfg));

// Readers — atomic load, ref-count handled correctly
auto cfg = config.load();
use(*cfg);
```

Before C++20, `std::atomic_load/store` free functions existed but were deprecated. This is the correct modern form. The `shared_ptr`'s ref-count bumps are atomic — no external mutex needed.

### `std::atomic::wait()` / `notify_one()` / `notify_all()` (C++20)

Atomic variables can now block efficiently (like a semaphore but tied to a value):

```cpp
std::atomic<int> flag{0};

// Thread A — waits until flag != 0
flag.wait(0);  // blocks without spin if flag == 0
// ... continue when flag changes ...

// Thread B
flag.store(1);
flag.notify_one();  // unblock one waiter
```

Replaces spin-polling `while (flag.load() == 0);` which wastes CPU. The implementation uses OS futex on Linux — efficient blocking.

---

## C++26 — The Frontier

### `std::hazard_pointer` (C++26, P2530)

Lock-free deferred reclamation for high-scale data structures. A thread "protects" a pointer by registering a hazard pointer. Memory cannot be freed while any hazard pointer references it.

```cpp
// Reader
auto hp = std::make_hazard_pointer(domain);
Node* node = hp.protect(head);  // atomic load + hazard registration
use(*node);                     // safe — cannot be freed while hp held
hp.reset();                     // release hazard — memory can now be freed

// Writer / deleter
rcu_retire(old_node);  // deferred free — waits until no hazard pointers reference it
```

**vs `shared_ptr`:** No atomic ref-count bump on the read path. Readers don't pay for writers' allocations. Scales far better under many concurrent readers.

**Use for:** Lock-free linked lists, queues, hash maps where readers must not be blocked or slowed by writers.

### `std::rcu_domain` / RCU (C++26, P2545) — Read-Copy-Update

From the Linux kernel (in production since 2002). Near-zero overhead reads. Writers copy the object, modify the copy, publish atomically. Old copies freed when all readers are done.

```cpp
// Reader — extremely cheap, no lock, no barrier on x86
{
    std::shared_lock<std::rcu_domain> guard(rcu_default_domain());
    auto* ptr = rcu_ptr.load(std::memory_order_consume);
    use(*ptr);
}   // guard destructor marks this reader as done

// Writer
auto* new_data = new Data(*current_ptr);
new_data->value = updated_value;
rcu_ptr.store(new_data, std::memory_order_release);
rcu_retire(old_ptr);  // freed when all pre-existing readers finish
```

**RCU readers never block** — not even when a writer is active. Compare to `shared_mutex` where a write-lock acquisition must wait for all readers to finish. With RCU, writers proceed while old readers hold old copies; readers proceed without any lock.

**When to use:** Almost-never-written data accessed from many threads — routing tables, firewall rules, policy objects, DNS caches, feature flag tables.

**Rule:** Hazard pointers ≈ scalable reference counting (multiple objects, fine-grained). RCU ≈ scalable readers-writer lock without any reader-side lock at all (one or few objects, coarse access).

### `std::execution` — Senders/Receivers (C++26, P2300)

Composable, lazy, structured asynchronous work graphs. Adopted for C++26.

```cpp
#include <execution>
namespace ex = std::execution;

auto work = ex::just(42)
    | ex::then([](int x) { return x * 2; })
    | ex::then([](int x) { return std::to_string(x); })
    | ex::on(thread_pool.get_scheduler());

auto [result] = ex::sync_wait(work).value();
// result == "84"
```

**Core concepts:**

| Concept | Role |
| --- | --- |
| **Sender** | Describes future work. Lazy — nothing runs until connected. |
| **Receiver** | Consumes the result: value, error, or cancellation signal. |
| **Scheduler** | Where work runs — thread pool, GPU, inline, I/O. |
| **Algorithms** | `just`, `then`, `when_all`, `bulk`, `on`, `transfer`, `let_value`, `split` |

**Why it matters:** Replaces ad-hoc `future`/`promise`/`thread` compositions which leak threads and have no structured cancellation. With senders/receivers, cancellation propagates automatically through the entire work graph. No detached threads. Works across CPUs and GPUs (NVIDIA stdexec reference implementation).

---

## What C++20 Made Obsolete

| Old pattern | Modern replacement |
| --- | --- |
| `std::atomic_load(shared_ptr*)` (free functions) | `std::atomic<shared_ptr<T>>` specialization |
| Spin-polling `while (!flag.load())` | `flag.wait(false)` + `notify_one()` (C++20) |
| Manual `t.join()` in destructor | `std::jthread` auto-join |
| Manual stop flag `atomic<bool>` + check loop | `std::stop_token` + `jthread` |
| Platform semaphores (`sem_t`, Windows HANDLE) | `std::counting_semaphore` |
| Manual countdown with atomics | `std::latch` |
| Thread barrier with mutex + condition_variable | `std::barrier` |
| Atomic operations on borrowed pointers (hacks) | `std::atomic_ref<T>` |

---

## Java vs C++ Concurrency — Side-by-Side

| Aspect | Java | C++ |
| --- | --- | --- |
| Basic lock | `synchronized` — every object has a hidden monitor | `std::mutex` + RAII wrapper — explicit, zero hidden cost |
| Recursive lock | Always reentrant (you pay even when you don't need it) | Explicit `recursive_mutex` — pay only when needed |
| Readers-writer lock | `ReentrantReadWriteLock` — verbose | `shared_mutex` + `shared_lock` / `unique_lock` — concise |
| Condition variable | `wait()` / `notify()` on any object — one wait set per object | `condition_variable` — explicit, separate from mutex |
| Thread cancellation | `Thread.interrupt()` — throws exception, invasive | `stop_token` — cooperative poll, thread controls exit point |
| Atomic ordering | `volatile` = always `seq_cst`, no choice | 6 memory orders — tune to actual causality requirement |
| Safe shared pointer | GC handles it | `std::atomic<shared_ptr<T>>` (C++20) |
| Lock-free reclamation | GC | Hazard pointers / RCU (C++26) |
| Async composition | `CompletableFuture` | `std::execution` senders/receivers (C++26) |
| Object overhead | 2-word monitor header on every object | Zero — mutex is a separate object you place explicitly |

---

## Thread Contention — What It Is and How It Happens in C++

**Contention:** Multiple threads racing for the same mutex or atomic. Blocked threads are descheduled by the OS. Cost: ~10,000–100,000 ns per context switch.

**Symptoms:**
- Adding more threads doesn't improve (or worsens) throughput
- High CPU utilization but low actual work rate
- Profiler shows threads spending most time in `pthread_mutex_lock` or `futex`

**How to measure:**
- **perf** on Linux: `perf lock record ./program` then `perf lock report`
- **Valgrind Helgrind / DRD:** detect data races and lock contention patterns
- **ThreadSanitizer (TSan):** compile with `-fsanitize=thread` — detects races at runtime

**Contention behavior per primitive:**

| Primitive | Under Contention |
| --- | --- |
| `std::mutex` | OS wait → context switch (~10k–100k ns) |
| `atomic<T>` CAS | CPU spin and retry — burns CPU, no context switch |
| `shared_mutex` read lock | CAS on shared counter — cache-line ping-pong under many readers |
| `counting_semaphore` | OS efficient blocking (futex) |
| `atomic::wait()` | OS efficient blocking (futex) — no busy spin |
| Optimistic CAS loop | Proportional to contention — degrades under extreme load |

**Fine-grained locking pattern:**

```cpp
// Naive: all threads contend on one lock
std::mutex global_mtx;
void update(int key, int val) {
    std::lock_guard<std::mutex> lk(global_mtx);
    map[key] = val;
}

// Striped: only threads touching the same bucket contend
static constexpr int STRIPES = 16;
std::mutex locks[STRIPES];

void update(int key, int val) {
    int stripe = std::hash<int>{}(key) % STRIPES;
    std::lock_guard<std::mutex> lk(locks[stripe]);
    segments[stripe][key] = val;
}
```

---

## Full Decision Matrix

```
Protect shared data, simple scope
  → lock_guard<mutex>

Protect shared data, need condition variable
  → unique_lock<mutex> + condition_variable

Lock two or more mutexes atomically without deadlock
  → scoped_lock(m1, m2)

Read-heavy data, rare writes
  → shared_mutex + shared_lock (readers) + unique_lock (writers)

Same class methods need to re-enter mutex
  → recursive_mutex (prefer redesign first)

Simple counter or flag, no compound op
  → atomic<T> — choose memory_order by causality:
      independent stats → relaxed
      publish/subscribe → release + acquire
      unsure → seq_cst

Block until a value changes (no spin)
  → atomic<T>::wait() + notify_one/all (C++20)

Signal between threads, no spurious wakeups
  → binary_semaphore

Limit concurrent resource access to N
  → counting_semaphore<N>

Wait for N tasks to finish, one-shot
  → latch

Repeated phased parallel work
  → barrier

Background thread, auto-join, cooperative cancellation
  → jthread + stop_token

Atomic on existing non-atomic object
  → atomic_ref<T>

Lock-free atomic config/snapshot reload
  → atomic<shared_ptr<T>>

Lock-free data structure, scalable reference
  → C++26 hazard_pointer

Near-zero-cost reads of almost-static data
  → C++26 RCU

Composable async pipelines, structured cancellation
  → C++26 std::execution (senders/receivers)
```

---

## Full Reader-Writer Pattern (Parking Lot Analogy)

Concrete example — thread-safe cache with three access tiers:

```cpp
class ParkingCollection {
    mutable std::shared_mutex rw_;
    std::unordered_map<int, ParkingSpot> available_;
    std::unordered_map<int, ParkingSpot> occupied_;

public:
    // Multiple readers can check availability simultaneously
    bool isAvailable(int spotId) const {
        std::shared_lock<std::shared_mutex> lk(rw_);
        return available_.count(spotId) > 0;
    }

    // Exclusive write — assign spot to vehicle
    bool assign(int spotId, int vehicleId) {
        std::unique_lock<std::shared_mutex> lk(rw_);
        auto it = available_.find(spotId);
        if (it == available_.end()) return false;
        occupied_[vehicleId] = it->second;
        available_.erase(it);
        return true;
    }

    // Release spot — exclusive write
    void release(int vehicleId) {
        std::unique_lock<std::shared_mutex> lk(rw_);
        auto it = occupied_.find(vehicleId);
        if (it != occupied_.end()) {
            available_[it->second.id] = it->second;
            occupied_.erase(it);
        }
    }
};
```

For higher scale — per-spot `std::mutex` with `try_lock`:

```cpp
class ParkingSpot {
    std::mutex lock_;
    bool available_ = true;

public:
    // tryAssign returns false immediately if another thread is mid-assignment
    bool tryAssign(int vehicleId) {
        std::unique_lock<std::mutex> lk(lock_, std::try_to_lock);
        if (!lk.owns_lock()) return false;  // another thread holds it
        if (!available_) return false;
        available_ = false;
        return true;
    }

    void release() {
        std::lock_guard<std::mutex> lk(lock_);
        available_ = true;
    }
};
```

`std::try_to_lock` tag tells `unique_lock` to attempt lock without blocking. If it fails, `owns_lock()` returns false. This is the C++ equivalent of Java's `ReentrantLock::tryLock()`.
