# Solution Guide — Chat Room

**Read after your attempt.**

---

## The Mediator Insight

Without Mediator: every user holds references to every other user. N users = N² references. Adding/removing a user requires updating O(N) peers. Tight coupling.

With Mediator: every user holds ONE reference (to ChatRoom). N users = N references total. Adding/removing is O(1) in the mediator's list. Loose coupling.

---

## Entity Map

| Class/Interface | Type | Responsibility |
|----------------|------|---------------|
| `ChatMediator` | Interface | Declares `addUser`, `removeUser`, `broadcast`, `directMessage`, `getHistory` |
| `ChatRoom` | Concrete Mediator | Holds `CopyOnWriteArrayList<User>` + message history; routes all communication |
| `User` | Abstract Colleague | Holds `ChatMediator mediator`; never holds other `User` refs |
| `ChatUser` | Concrete Colleague | Implements `send` (→ `mediator.broadcast`) and `receive` |
| `Message` | Immutable value | senderId, content, timestamp, type, optional recipientId |
| `ChatRoomManager` | Registry | `Map<String, ChatRoom>`; creates and retrieves rooms |

---

## Full Implementation

```java
// Message — immutable
public final class Message {
    public enum Type { BROADCAST, DIRECT }
    private final String senderId;
    private final String content;
    private final Instant timestamp = Instant.now();
    private final Type type;
    private final String recipientId; // null for broadcast

    public Message(String senderId, String content, Type type, String recipientId) {
        this.senderId = senderId; this.content = content;
        this.type = type; this.recipientId = recipientId;
    }
    public String getSenderId()    { return senderId; }
    public String getContent()     { return content; }
    public Type getType()          { return type; }
    public String getRecipientId() { return recipientId; }
    @Override public String toString() {
        return String.format("[%s] %s: %s", timestamp, senderId, content);
    }
}

// Mediator interface
public interface ChatMediator {
    void addUser(User user);
    void removeUser(User user);
    void broadcast(String content, User sender);
    void directMessage(String content, User sender, String recipientId);
    List<Message> getHistory();
}

// Concrete Mediator
public class ChatRoom implements ChatMediator {
    private final String name;
    // CopyOnWriteArrayList: safe concurrent iteration during broadcast
    private final List<User> users    = new CopyOnWriteArrayList<>();
    private final List<Message> history = new CopyOnWriteArrayList<>();

    public ChatRoom(String name) { this.name = name; }

    @Override public void addUser(User user) {
        users.add(user);
    }

    @Override public void removeUser(User user) {
        users.remove(user);
    }

    @Override public void broadcast(String content, User sender) {
        Message msg = new Message(sender.getId(), content, Message.Type.BROADCAST, null);
        history.add(msg);
        for (User user : users) {
            if (!user.getId().equals(sender.getId())) { // no self-echo
                user.receive(msg);
            }
        }
    }

    @Override public void directMessage(String content, User sender, String recipientId) {
        Message msg = new Message(sender.getId(), content, Message.Type.DIRECT, recipientId);
        history.add(msg);
        for (User user : users) {
            if (user.getId().equals(recipientId)) {
                user.receive(msg);
                return;
            }
        }
        System.out.println("User " + recipientId + " not in room " + name);
    }

    @Override public List<Message> getHistory() {
        return Collections.unmodifiableList(history);
    }
}

// Abstract Colleague — depends on mediator interface, not concrete ChatRoom
public abstract class User {
    protected final ChatMediator mediator;
    protected final String id;
    protected final String name;

    public User(ChatMediator mediator, String id, String name) {
        this.mediator = mediator; this.id = id; this.name = name;
    }

    public abstract void send(String message);
    public abstract void sendDirect(String message, String recipientId);
    public abstract void receive(Message message);

    public String getId()   { return id; }
    public String getName() { return name; }
}

// Concrete Colleague — ZERO direct references to other Users
public class ChatUser extends User {
    public ChatUser(ChatMediator mediator, String id, String name) {
        super(mediator, id, name);
    }

    @Override public void send(String message) {
        mediator.broadcast(message, this);
    }

    @Override public void sendDirect(String message, String recipientId) {
        mediator.directMessage(message, this, recipientId);
    }

    @Override public void receive(Message message) {
        System.out.println(name + " received [" + message.getType() + "] from "
            + message.getSenderId() + ": " + message.getContent());
    }
}

// ChatRoomManager — manages multiple rooms
public class ChatRoomManager {
    private final Map<String, ChatRoom> rooms = new ConcurrentHashMap<>();

    public ChatRoom getOrCreate(String roomName) {
        return rooms.computeIfAbsent(roomName, ChatRoom::new);
    }

    public Optional<ChatRoom> getRoom(String name) {
        return Optional.ofNullable(rooms.get(name));
    }
}
```

---

## Multi-Room User

```java
// User can be in multiple rooms — holds Map<roomId, mediator>
public class MultiRoomChatUser extends User {
    private final Map<String, ChatMediator> rooms = new ConcurrentHashMap<>();

    public void joinRoom(String roomId, ChatMediator room) {
        rooms.put(roomId, room);
        room.addUser(this);
    }

    public void leaveRoom(String roomId) {
        ChatMediator room = rooms.remove(roomId);
        if (room != null) room.removeUser(this);
    }

    public void sendToRoom(String roomId, String message) {
        ChatMediator room = rooms.get(roomId);
        if (room != null) room.broadcast(message, this);
    }
    // ... other methods
}
```

---

## Mediator vs Observer

A common interview question. Key difference:

| | Mediator | Observer |
|---|---|---|
| Relationship | Many-to-many coordination | One-to-many notification |
| Who knows whom | Mediator knows all colleagues | Subject knows observers |
| Purpose | Reduce direct coupling between peers | React to state change |
| Chat example | ChatRoom coordinates users | Order status notifies subscribers |

---

## Extension Points

| "Now add..." | Change |
|-------------|--------|
| Multi-room support | User holds `Map<String, ChatMediator>` |
| Message types (image, file) | `Message` becomes abstract base; `TextMessage`, `ImageMessage` extend it |
| Offline queue | `User` has `Queue<Message> offlineQueue`; drained on reconnect |
| Admin/moderation | `AdminChatRoom extends ChatRoom`; `mute(userId)` checks before deliver |
| Read receipts | `Message` gets `Set<String> readByUserIds`; `User.receive()` calls back |

---

## What Strong Candidates Do Differently

- `ChatMediator` as interface (not direct `ChatRoom` class dependency) — DIP
- `CopyOnWriteArrayList` for users — explains: broadcast iterates while join/leave can happen
- No self-echo in `broadcast()` — explicitly checked and explained
- `Message` as immutable value object with `Instant.now()` captured at construction
- User holds `ChatMediator`, never `ChatUser` — the pattern's whole point stated explicitly

## What Average Candidates Miss

- Users hold `List<User> contacts` for direct messaging — defeats the pattern
- `ChatRoom` as concrete dependency in User (violates DIP)
- `ArrayList` for users — `ConcurrentModificationException` under concurrency
- Self-echo not considered — sender receives their own broadcast
- No `ChatMediator` interface — single concrete `ChatRoom` class, untestable
