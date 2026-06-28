# Lessons Learned — LRU Cache

---

## Mistake 1: Not Storing Key in Node

**What happens:** Candidate creates `Node(value, prev, next)` without key. When evicting tail node, can't remove it from HashMap.

**Why it's wrong:** `HashMap.remove()` needs the key, not the node. If the node doesn't carry its key, you have to scan the HashMap to find it — O(n).

**Fix:** Always store key in Node:
```java
class Node {
    int key, value; // key is required for HashMap eviction
    Node prev, next;
}
// On eviction:
Node lru = dll.removeTail();
cache.remove(lru.key); // O(1) because key is in the node
```

**Gap dimension:** Entity Modeling (Dimension 2)

---

## Mistake 2: Not Using Sentinel Nodes

**What happens:** DLL implemented without dummy head/tail. Every add/remove has null checks: "if head is null, if tail is null, if node.prev is null..."

**Why it's wrong:** Extra conditional complexity, easy to introduce bugs at boundary cases (empty list, single node).

**Fix:** Use sentinel (dummy) nodes:
```java
DoublyLinkedList() {
    head = new Node(0, 0); // dummy
    tail = new Node(0, 0); // dummy
    head.next = tail;
    tail.prev = head;
}
// Now remove() never needs null checks:
void remove(Node node) {
    node.prev.next = node.next;
    node.next.prev = node.prev;
}
// Head and tail sentinels always exist — no edge cases
```

**Gap dimension:** Entity Modeling (Dimension 2)

---

## Mistake 3: Using Synchronized on All Methods Without Discussion

**What happens:** Candidate slaps `synchronized` on `get()` and `put()` without explaining tradeoffs. Works correctly but misses an interview signal.

**Why it's a gap:** Reads don't need exclusive access unless they also promote the node. `ReentrantReadWriteLock` allows concurrent reads — better throughput under read-heavy workload.

**Better answer:**
```
"I'll use synchronized for simplicity — it's correct. The more
performant option is ReadWriteLock: concurrent reads, exclusive
write. get() that hits a miss is pure read. get() that hits
and promotes needs a write lock. put() always needs write."
```

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 4: Using Java's LinkedHashMap Without Explaining It

**What happens:** Candidate reaches for `new LinkedHashMap<>(capacity, 0.75f, true)` with `removeEldestEntry` override.

**Why it matters:** This is a legitimate shortcut, but interviewer will immediately ask "how does LinkedHashMap implement this?" If you can't explain, you've used a black box.

**Acceptable response:**
> "Java's LinkedHashMap is basically this: an access-ordered doubly linked list backed by a HashMap. I can implement it directly, or use LinkedHashMap as the foundation. Which do you prefer?"

**Gap dimension:** Communication (Dimension 7) — fine to use library, but must explain internals.

---

## Problem-Specific Gotchas

- `get()` promotes the item to most recently used — many candidates forget this
- `put()` on existing key should update value AND promote to head
- Capacity check: `size >= capacity` triggers eviction BEFORE adding new node
- DLL size tracking: increment on addToHead, decrement on remove

---

## Self-Assessment Checklist

- [ ] Does `get()` promote the retrieved node to head?
- [ ] Does `put()` on existing key update value AND promote to head?
- [ ] Does Node store the key (not just value)?
- [ ] Do sentinel nodes eliminate null checks in DLL?
- [ ] Is eviction correct: remove tail, then remove tail's key from HashMap?
- [ ] Is concurrency addressed (synchronized or ReadWriteLock)?
- [ ] Can generics be added without structural change?

---

## Remediation Targets

| Dimension | If you scored 0–1 | Go to |
|-----------|------------------|-------|
| Entity Modeling | Node missing key, no sentinels | GAP_REMEDIATION.md#gap-2 — think about what each object needs to DO |
| Concurrency | No locking discussed | GAP_REMEDIATION.md#gap-6 — shared state audit: what's mutated? |
| Communication | Used LinkedHashMap as black box | GAP_REMEDIATION.md#gap-7 — explain internals when using shortcuts |
