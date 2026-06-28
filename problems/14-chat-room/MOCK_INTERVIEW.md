# Mock Interview Script — Chat Room

**For the interviewer.**

---

## Opening

> "Design a group chat room application. Users can join rooms and send messages to all members. No user should communicate directly with another user — all routing goes through the room. Ask clarifying questions first."

| If candidate asks | Answer |
|------------------|--------|
| Broadcast to sender too? | No — sender doesn't receive their own message |
| Direct messages? | Yes — DM to a specific user in the same room |
| Multiple rooms? | Yes — a user can be in multiple rooms |
| Message history? | Yes — in-memory per room |
| Thread safety? | Yes — concurrent messages from different users |
| Online/offline? | In-memory only — assume all users are always online |

---

## Phase 2: Core Design — What to Watch For

**The defining signal:** Do users hold references to other users?

Bad: `ChatUser` has a `Map<String, User> contacts` to message directly.

Good: `ChatUser.send(msg)` calls `mediator.broadcast(msg, this)`. User holds ONLY a `ChatMediator` reference — never another `User`.

If users hold direct references:
> "What happens when a user leaves the room? How many places need to be updated if users hold direct references to each other?"

**Interface signal:** Does candidate define `ChatMediator` as an interface or go straight to `ChatRoom` class?

Good: `User` depends on `ChatMediator` interface, not `ChatRoom` — satisfies DIP, enables testing with a mock mediator.

---

## Extension Probe (~35 min)

> "Now add multiple chat rooms. A user can be in 'general', 'engineering', and 'random' simultaneously."

Strong: User holds a `Map<String, ChatMediator> rooms`. `send(roomId, message)` dispatches to the right mediator. `ChatRoomManager` manages the registry. Zero changes to `ChatRoom` or `ChatUser`.

> "Now add message types — text, image, file share. How does your design change?"

Strong: `Message` becomes a class hierarchy: `TextMessage`, `ImageMessage`, `FileMessage` — all extend `Message`. `ChatRoom.broadcast()` accepts `Message`. Zero changes to User or ChatRoom routing logic.

---

## Concurrency Probe (~40 min)

> "Alice and Bob send messages to the same room simultaneously. What breaks?"

Strong: Concurrent writes to the `users` list during iteration in `broadcast()` → `ConcurrentModificationException`. Fix: `CopyOnWriteArrayList` for users. History list also needs protection: `CopyOnWriteArrayList` or `synchronized`. Mentions: `addUser`/`removeUser` concurrent with `broadcast` is the core race.

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | No self-echo, DM support, multi-room, history | |
| Entity Modeling | `ChatMediator` interface; User holds mediator not peers | |
| Design Patterns | Mediator: users communicate ONLY through room | |
| SOLID | DIP: User depends on ChatMediator interface; OCP: new message type = new class | |
| Extensibility | Multi-room absorbed; new message type absorbed | |
| Concurrency | `CopyOnWriteArrayList` for users; history thread-safe | |
| Communication | Explained WHY mediator (decoupling), WHY interface over concrete class | |
| **TOTAL** | | **/21** |
