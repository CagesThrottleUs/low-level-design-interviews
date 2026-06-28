# Lessons Learned — Chat Room

---

## Mistake 1: Users Hold Direct References to Each Other

**What happens:** `ChatUser` has `Map<String, User> contacts`. Sends directly via `contacts.get(recipientId).receive(msg)`.

**Why it's wrong:** This defeats the entire Mediator pattern. When a user leaves, you must update every other user's contact map. N users = N² references to maintain. Adding a new message type requires changing every User class.

**Fix:** `ChatUser` holds ONLY `ChatMediator mediator`. Every send goes through `mediator.broadcast()` or `mediator.directMessage()`. Zero user-to-user references.

**Gap dimension:** Design Patterns (Dimension 3)

---

## Mistake 2: `ChatRoom` as Direct Dependency (Not Interface)

**What happens:**
```java
public abstract class User {
    protected ChatRoom room; // concrete dependency
}
```

**Why it's wrong:** User is coupled to `ChatRoom`. Can't test User with a mock. Can't swap to a different mediator implementation. Violates DIP.

**Fix:**
```java
public abstract class User {
    protected ChatMediator mediator; // depends on interface
}
```
`ChatUser` can now be unit-tested by injecting a mock `ChatMediator`.

**Gap dimension:** SOLID — DIP (Dimension 4)

---

## Mistake 3: ArrayList for Users — ConcurrentModificationException

**What happens:** `List<User> users = new ArrayList<>()`. `broadcast()` iterates users. Concurrently, another thread calls `addUser()`. `ConcurrentModificationException` thrown mid-broadcast.

**Fix:** `CopyOnWriteArrayList<User>` — iteration works on a snapshot. `addUser`/`removeUser` copies the array (acceptable: joins/leaves are rare vs message sends).

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 4: Self-Echo in Broadcast

**What happens:** Sender receives their own broadcast message.

**Why it's a gap:** Explicitly asked about in requirements ("does sender receive their own message?"). Forgetting it means the requirements phase was skipped or incomplete.

**Fix:**
```java
for (User user : users) {
    if (!user.getId().equals(sender.getId())) { // skip sender
        user.receive(msg);
    }
}
```

**Gap dimension:** Requirements (Dimension 1)

---

## Mistake 5: Mediator Becomes a God Object

**What happens:** `ChatRoom` accumulates: spam detection, authentication, message translation, analytics, notification delivery, message encryption — all inside `broadcast()`.

**Why it's wrong:** Mediator should route only. Each concern belongs in its own class. Spam detection = `SpamFilter` Strategy. Encryption = decorator wrapping the mediator.

**Fix:** Keep `ChatRoom` focused on routing. Apply filters/decorators at the point of registration or via Strategy.

**Gap dimension:** SOLID — SRP (Dimension 4)

---

## Self-Assessment Checklist

- [ ] Does `ChatUser` hold ONLY `ChatMediator mediator` (not `Map<String, User>`)?
- [ ] Is `ChatMediator` an interface (not a concrete `ChatRoom` dependency)?
- [ ] Is the users list `CopyOnWriteArrayList` (not `ArrayList`)?
- [ ] Does `broadcast()` skip the sender (no self-echo)?
- [ ] Is `Message` immutable (all fields final, `Instant.now()` at construction)?
- [ ] Can a new message type be added without changing ChatRoom?
- [ ] Does `ChatRoom` stay focused on routing (not accumulating business logic)?

---

## Mediator vs Observer Quick Distinction

| Mediator | Observer |
|----------|----------|
| Many-to-many coordination | One-to-many notification |
| ChatRoom routes user messages | Order notifies status subscribers |
| Mediator knows all colleagues | Subject knows observers |
| Reduces N² connections to N | Decouples event source from handlers |

---

## Remediation Targets

| Dimension | If scored 0–1 | Go to |
|-----------|--------------|-------|
| Design Patterns | Users hold direct refs | GAP_REMEDIATION.md#gap-3 — Mediator trigger: "N components need to communicate" |
| SOLID | Concrete ChatRoom dependency | GAP_REMEDIATION.md#gap-4 — DIP violation |
| Concurrency | ArrayList, not CopyOnWriteArrayList | GAP_REMEDIATION.md#gap-6 |
| Requirements | Self-echo not handled | GAP_REMEDIATION.md#gap-1 |
