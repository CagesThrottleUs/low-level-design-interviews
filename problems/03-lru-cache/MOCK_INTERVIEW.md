# Mock Interview Script — LRU Cache

**For the interviewer.**

---

## Opening

> "Design an LRU (Least Recently Used) cache. It should support get and put operations, both in O(1) time. Ask any clarifying questions first."

| If candidate asks | Answer |
|------------------|--------|
| Key/value types? | Integer keys and values; generics are a bonus |
| Thread safety? | Yes — concurrent get and put |
| Cache miss behavior? | Return -1 |
| Update existing key? | Yes — put updates value and marks as recently used |
| Eviction callback? | Not required for MVP |

---

## Phase 2: Core Design

**Key watch:** How quickly does the candidate arrive at HashMap + DoublyLinkedList?

**Progression to watch for:**
1. Candidate asks about O(1) requirement ✓
2. Considers HashMap (O(1) lookup) but asks "how to track order?" ✓
3. Arrives at DLL for order tracking ✓
4. Realizes Node must be in both structures ✓
5. Realizes Node must store key (for HashMap removal on eviction) ✓

If stuck at step 2 (needs order tracking):
> "If you needed to always know which item was used least recently, in O(1), what data structure comes to mind?"

If stuck at step 5 (forgetting key in Node):
> "When you evict the tail from the linked list, how do you know which key to remove from the HashMap?"

---

## Extension Probe (~35 min)

> "How would you make this support generic key and value types instead of just integers?"

Strong: Straightforward — `class LRUCache<K, V>` and `class Node<K, V>`. No structural change.

> "Now add an eviction listener — a callback that fires when an item is evicted."

Strong: Adds `EvictionListener` interface, calls it in `removeTail()`. New listener = new class.

---

## Concurrency Probe (if not raised)

> "What happens if two threads call put() simultaneously?"

Strong: Identifies race on HashMap and DLL. Discusses `synchronized` vs `ReadWriteLock`. Notes that get-with-promotion also needs write lock, not read lock.

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | O(1), thread safety, cache miss behavior, update semantics | |
| Entity Modeling | Node (key+value+prev+next), DLL with sentinels, clean LRUCache | |
| Design Patterns | N/A for patterns, but may apply Strategy for eviction policy extension | |
| SOLID | Single-purpose classes: Node, DLL, Cache | |
| Extensibility | Generics, eviction listener absorbed cleanly | |
| Concurrency | ReadWriteLock or synchronized, explains tradeoff | |
| Communication | Explained WHY DLL+HashMap, WHY key in Node | |
| **TOTAL** | | **/21** |
