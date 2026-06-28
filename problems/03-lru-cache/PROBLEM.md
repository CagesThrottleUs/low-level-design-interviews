# LRU Cache

**Difficulty:** Foundation
**Category:** Data Structure + Concurrency
**Time Box:** 45 min (discussion) / 90 min (machine coding)
**Key Patterns:** No classic GoF pattern — this tests data structure design and thread safety
**Asked at:** Google, Meta, Amazon, Microsoft, LinkedIn, Bloomberg

---

## Problem Statement

Design a Least Recently Used (LRU) Cache with `get` and `put` operations. The cache has a fixed capacity. When the cache is full and a new item is added, the least recently used item is evicted. Both `get` and `put` should run in O(1) time.

The implementation must be thread-safe for concurrent reads and writes.

---

## Actors

| Actor | Description |
|-------|-------------|
| Caller | Any system or user that calls get() or put() |
| LRU Cache | Manages key-value storage with LRU eviction |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: `get(key)` — returns value if key exists; returns -1 (or null) if not. Marks key as recently used.
- FR2: `put(key, value)` — inserts or updates. If capacity exceeded, evicts LRU item first.
- FR3: Both operations are O(1)
- FR4: Thread-safe for concurrent access

**Secondary (implement if time allows):**
- FR5: `delete(key)` — explicit removal
- FR6: Configurable eviction listener (callback when item evicted)
- FR7: TTL (time-to-live) per entry
- FR8: Typed generics instead of raw Object

---

## Non-Functional Requirements

- **Concurrency:** Multiple threads calling get/put simultaneously
- **Time complexity:** O(1) for get and put — hard requirement
- **Capacity:** Fixed at construction time

---

## Constraints and Assumptions

- Integer keys and values for simplicity (generics are a bonus)
- Return -1 for cache miss on `get`
- Capacity >= 1
- Thread safety required: reads and writes can be concurrent
- Out of scope: distributed caching, persistence, TTL (unless extra time)

---

## Good Clarifying Questions to Ask

1. What types for keys and values? (int/int or generic K/V?)
2. Thread safety required?
3. What to return on cache miss — null, -1, Optional?
4. What happens if `put` is called with same key — update or ignore?
5. Should `get` promote an item to most-recently-used?
6. Any eviction callbacks or listeners?

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **LRUCache** — public API; holds HashMap + DoublyLinkedList; coordinates all operations
- **DoublyLinkedList** — maintains order (most recent at head, LRU at tail); O(1) move and remove
- **Node** — holds key + value; prev + next pointers; HashMap value so we can locate by key
- **HashMap** — key → Node mapping; O(1) lookup by key

**Why doubly linked list + hashmap?**
- HashMap gives O(1) lookup by key
- DoublyLinkedList gives O(1) reorder (move to head) and O(1) eviction (remove tail)
- Node in both → O(1) for all operations

</details>
