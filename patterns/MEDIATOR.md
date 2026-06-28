# Mediator Pattern

**Category:** Behavioral
**LLD interview frequency:** Tier 2
**Problem:** 14 — Chat Room / Group Messaging

---

## The Problem It Solves

When N components all need to communicate with each other, you get O(N²) dependencies. Adding one component means updating N-1 others. Removing one requires cleaning up N-1 references.

```java
// Without Mediator — N² references
class ChatUser {
    private List<ChatUser> contacts; // every user knows every other user

    public void sendMessage(String msg) {
        for (ChatUser contact : contacts) {
            contact.receive(msg); // direct coupling
        }
    }
    // Adding a new user: update EVERY existing user's contact list
    // Removing a user: update EVERY existing user's contact list
}
```

---

## Intent

Define an object that encapsulates how a set of objects interact. Promotes loose coupling by preventing objects from referring to each other explicitly. Reduces N² connections to N connections.

---

## Trigger Signals

Apply Mediator when you see:
- "Multiple components need to communicate, but they shouldn't know about each other"
- "Adding or removing a component should not require updating other components"
- "All routing/coordination logic is scattered across many classes"
- Problems: chat rooms, air traffic control, stock exchange, UI form controllers

---

## Structure

```java
// Mediator interface — all colleagues depend on this, not on each other
public interface ChatMediator {
    void addUser(User user);
    void removeUser(User user);
    void broadcast(String content, User sender);
    void directMessage(String content, User sender, String recipientId);
}

// Concrete Mediator — knows about everyone; coordinates all interactions
public class ChatRoom implements ChatMediator {
    // CopyOnWriteArrayList: safe for concurrent broadcast + join/leave
    private final List<User> users = new CopyOnWriteArrayList<>();
    private final List<Message> history = new CopyOnWriteArrayList<>();

    @Override
    public void broadcast(String content, User sender) {
        Message msg = new Message(sender.getId(), content, Message.Type.BROADCAST, null);
        history.add(msg);
        for (User user : users) {
            if (!user.getId().equals(sender.getId())) {
                user.receive(msg); // mediator knows all users; users don't know each other
            }
        }
    }
}

// Colleague — depends ONLY on ChatMediator interface, never on other users
public class ChatUser extends User {
    public ChatUser(ChatMediator mediator, String id, String name) {
        super(mediator, id, name);
    }

    @Override public void send(String message) {
        mediator.broadcast(message, this); // sends TO mediator, not to users
    }

    @Override public void receive(Message message) {
        System.out.println(name + " received: " + message.getContent());
    }
}
```

---

## N connections vs N² connections

```
Without Mediator (5 users):
Alice ←→ Bob ←→ Carol ←→ Dave ←→ Eve
= 5×4/2 = 10 bidirectional references

With Mediator (5 users):
Alice → ChatRoom ← Bob ← Carol ← Dave ← Eve
= 5 references total

Adding Frank:
Without: update 5 existing users' contact lists
With: Frank registers with ChatRoom — zero existing changes
```

---

## Mediator vs Observer

Frequently confused. Know the distinction:

| | Mediator | Observer |
| --- | --- | --- |
| Relationship | Many-to-many coordination | One-to-many notification |
| Who knows whom | Mediator knows all colleagues | Subject knows observers |
| Direction | Colleagues → Mediator → Colleagues | Subject → Observers |
| Use when | Peers need to communicate with each other | Components react to state changes |
| Example | Chat room, air traffic control | Order status notifications, event bus |

---

## Underlying Principle

**SRP:** Coordination logic lives in the mediator, not in each component. Components only know their own business logic.
**DIP:** Colleagues depend on `ChatMediator` interface, not `ChatRoom` class. Enables mocking in tests.
**OCP:** Adding a new message type = new `Message` subclass. ChatRoom.broadcast() unchanged.

---

## Real-World Usage

- **Java's `ExecutorService`** — tasks don't communicate directly; executor mediates
- **Spring's `ApplicationEventPublisher`** — components publish events; Spring routes to listeners
- **MVC Controller** — Model and View communicate through Controller (a mediator)
- **Message brokers (Kafka, RabbitMQ)** — producers and consumers don't know about each other; broker mediates

---

## The God Mediator Trap

The Mediator pattern has one known failure mode: the mediator accumulates too much logic and becomes a God Object. When the mediator knows about authentication, spam detection, message formatting, analytics, and routing all at once — split it.

Fix: Keep the mediator focused on routing. Use Strategy for filtering, Decorator for logging, separate services for analytics.

---

## Common Mistakes

- Users hold direct references to each other (defeats the pattern entirely)
- `ChatRoom` as concrete dependency in User (violates DIP — use interface)
- `ArrayList` for users (ConcurrentModificationException during broadcast + join)
- Self-echo: sender receives their own broadcast

---

## When NOT to Use

- Only 2 components interacting — just let them reference each other
- Components don't need to interact at all — they're already decoupled
- The "mediator" would just be a thin wrapper doing nothing useful
