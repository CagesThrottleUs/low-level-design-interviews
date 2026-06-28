# Solution Guide — LRU Cache

**Read after your attempt.**

---

## Core Insight

O(1) for both get and put requires two data structures working together:

| Operation | HashMap alone | DLL alone | HashMap + DLL |
|-----------|--------------|-----------|---------------|
| Lookup by key | O(1) | O(n) | O(1) |
| Move to head (mark recent) | n/a | O(1) if node pointer known | O(1) |
| Evict LRU (tail) | O(1) but which key? | O(1) | O(1) |

**The trick:** Store the doubly linked list Node as the HashMap value. Given a key, we get the Node in O(1). Given the Node, we move or remove it in O(1).

---

## Entity Map

| Class | Responsibility |
|-------|---------------|
| `LRUCache` | Public API; owns HashMap + DLL; coordinates all ops |
| `Node` | Holds key + value + prev + next pointers |
| `DoublyLinkedList` | Maintains order; addToHead, remove, removeTail — all O(1) |
| `HashMap<Integer, Node>` | Key → Node mapping for O(1) lookup |

---

## Class Design

```java
class Node {
    int key, value;
    Node prev, next;
    Node(int key, int value) { this.key = key; this.value = value; }
}

class DoublyLinkedList {
    Node head, tail; // dummy sentinel nodes
    int size;

    DoublyLinkedList() {
        head = new Node(0, 0);
        tail = new Node(0, 0);
        head.next = tail;
        tail.prev = head;
    }

    void addToHead(Node node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
        size++;
    }

    void remove(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
        size--;
    }

    Node removeTail() {
        if (size == 0) return null;
        Node lru = tail.prev;
        remove(lru);
        return lru;
    }
}
```

---

## LRU Cache Implementation

```java
class LRUCache {
    private final int capacity;
    private final Map<Integer, Node> cache;
    private final DoublyLinkedList dll;
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new HashMap<>();
        this.dll = new DoublyLinkedList();
    }

    public int get(int key) {
        lock.readLock().lock();
        try {
            if (!cache.containsKey(key)) return -1;
        } finally {
            lock.readLock().unlock();
        }

        // Promote to head — requires write lock
        lock.writeLock().lock();
        try {
            Node node = cache.get(key);
            if (node == null) return -1; // evicted between locks
            dll.remove(node);
            dll.addToHead(node);
            return node.value;
        } finally {
            lock.writeLock().unlock();
        }
    }

    public void put(int key, int value) {
        lock.writeLock().lock();
        try {
            if (cache.containsKey(key)) {
                Node node = cache.get(key);
                node.value = value;
                dll.remove(node);
                dll.addToHead(node);
            } else {
                if (dll.size >= capacity) {
                    Node lru = dll.removeTail();
                    cache.remove(lru.key); // use key stored in node — why Node stores key
                }
                Node node = new Node(key, value);
                cache.put(key, node);
                dll.addToHead(node);
            }
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

---

## Concurrency Model

**Why `ReentrantReadWriteLock` over `synchronized`?**

- `synchronized` allows one thread at a time — even pure reads block each other
- `ReadWriteLock` allows multiple concurrent reads; exclusive for writes
- `get()` that hits cache and doesn't need promotion = pure read (concurrent OK)
- `get()` that requires LRU promotion = write (exclusive)
- `put()` = always write (exclusive)

**Alternative (simpler, less performant):** `synchronized` on all methods. Acceptable in interview if you explain the tradeoff.

**Why Node stores the key:**
When evicting the LRU node (tail), we need to remove it from the HashMap. The node is found via DLL, not via key — so the node must carry its own key for the `cache.remove(node.key)` call.

---

## Extension Points

| "Now add..." | Change |
|-------------|--------|
| Generic K/V types | `LRUCache<K,V>`, `Node<K,V>` — no structural change |
| Eviction listener | Add `EvictionListener` interface; call in `removeTail()` |
| TTL per entry | Add `expiresAt` to Node; background sweep or lazy expiry on `get` |
| LFU instead of LRU | Different eviction policy — Strategy pattern on eviction |

---

## What Strong Candidates Do Differently

- Draw the DLL + HashMap diagram before coding
- Use sentinel head/tail nodes to eliminate edge cases (empty list, single node)
- Store key in Node — immediately explains why when asked
- Discuss ReadWriteLock vs synchronized as a conscious tradeoff
- Handle the race in `get`: re-check after acquiring write lock

## What Average Candidates Miss

- Try to use only HashMap → can't do O(1) eviction order
- Try to use `LinkedHashMap` (Java's built-in) — acceptable shortcut, but must explain internals
- Don't store key in Node → can't remove from HashMap on eviction
- Use `synchronized` on all methods without mentioning ReadWriteLock alternative
- Don't use sentinel nodes → lots of null checks in DLL operations
