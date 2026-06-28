# Chat Room / Group Messaging

**Difficulty:** Intermediate
**Category:** Mediator Pattern
**Time Box:** 45 min (discussion) / 90 min (machine coding)
**Key Patterns:** Mediator (ChatRoom routes all communication), Observer (user receives messages)
**Asked at:** Amazon, Meta, Microsoft, Grab, Gojek

---

## Problem Statement

Design a group chat application. Users can join chat rooms and send messages. Messages are routed through a central chat room mediator — users never send messages directly to each other. The system supports broadcasting to all room members and direct messages to a specific user. A user can be a member of multiple rooms simultaneously.

This is the canonical Mediator pattern problem: components communicate through a mediator instead of holding direct references to each other.

---

## Actors

| Actor | Description |
|-------|-------------|
| User | Joins rooms; sends broadcast and direct messages; receives messages |
| ChatRoom (Mediator) | Routes messages; maintains member list; stores history |
| ChatRoomManager | Creates and manages multiple chat rooms |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: Users can join a chat room
- FR2: Users can leave a chat room
- FR3: A user sends a broadcast message to all other room members (not to themselves)
- FR4: A user sends a direct message to a specific room member by user ID
- FR5: User can be in multiple rooms simultaneously
- FR6: Chat room stores message history (in-memory)
- FR7: User's `receive()` is called when a message is delivered

**Secondary (implement if time allows):**
- FR8: Multiple chat rooms managed by a `ChatRoomManager`
- FR9: Offline message queue — messages stored if user is offline, delivered on reconnect
- FR10: Message reactions / read receipts

---

## Non-Functional Requirements

- **Decoupling:** Users must NOT hold direct references to other users
- **Extensibility:** New message type (image, file) must not require changing `ChatRoom`
- **Thread safety:** Concurrent messages from multiple users must not corrupt member list or history
- **Consistency:** Broadcast must go to all current members at time of send

---

## Constraints and Assumptions

- All users and rooms are in-memory
- User identified by a unique string ID
- Message is a string for MVP (typed messages are secondary)
- Out of scope: encryption, delivery guarantees, read receipts, user authentication

---

## Good Clarifying Questions to Ask

1. Broadcast to all members, or only a subset (channel topics)?
2. Can a user receive their own broadcast (echo)?
3. Should message history be per-room or per-user?
4. Is direct messaging within a room (DM to another room member), or global DM?
5. Can users be in multiple rooms simultaneously?
6. Online/offline status — does message delivery fail if user is offline?

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **ChatMediator** (interface) — declares: `addUser(user)`, `removeUser(user)`, `broadcast(message, sender)`, `directMessage(message, sender, recipientId)`, `getHistory()`
- **ChatRoom** (Concrete Mediator) — implements `ChatMediator`; holds `CopyOnWriteArrayList<User>` and `List<Message>`; routes all communication
- **User** (Abstract Colleague) — holds `ChatMediator mediator`; declares `send(message)`, `sendDirect(message, recipientId)`, `receive(message)`
- **ChatUser** (Concrete Colleague) — implements send/receive; has NO reference to other users
- **Message** — immutable: senderId, content, timestamp, type (BROADCAST/DIRECT), optional recipientId
- **ChatRoomManager** — creates and manages a `Map<String, ChatRoom>`

</details>
