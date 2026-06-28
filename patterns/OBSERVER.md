# Observer Pattern

**Category:** Behavioral
**LLD interview frequency:** Tier 1 — must know cold

---

## Intent

Define a one-to-many dependency between objects. When one object (subject) changes state, all its dependents (observers) are notified and updated automatically.

**One sentence:** Publisher knows nothing about subscribers; subscribers register to receive events.

---

## Trigger Signals

Apply Observer when you hear:
- "Multiple components need to react when X happens"
- "Notify all subscribers when [event] occurs"
- Problems: Notification System, Event Bus, Stock Ticker, Order Status Tracking, Pub/Sub

---

## Structure

```java
// 1. Observer interface
interface OrderObserver {
    void onStatusChange(Order order, OrderStatus newStatus);
}

// 2. Concrete observers
class EmailNotifier implements OrderObserver {
    public void onStatusChange(Order order, OrderStatus status) {
        sendEmail(order.getCustomerEmail(), "Order " + order.getId() + " is now " + status);
    }
}

class SMSNotifier implements OrderObserver {
    public void onStatusChange(Order order, OrderStatus status) {
        sendSMS(order.getCustomerPhone(), "Status: " + status);
    }
}

class AnalyticsDashboard implements OrderObserver {
    public void onStatusChange(Order order, OrderStatus status) {
        metrics.record("order.status_change", status.name());
    }
}

// 3. Subject (Observable)
class Order {
    private final List<OrderObserver> observers = new ArrayList<>();
    private OrderStatus status;

    public void addObserver(OrderObserver o) { observers.add(o); }
    public void removeObserver(OrderObserver o) { observers.remove(o); }

    public void updateStatus(OrderStatus newStatus) {
        this.status = newStatus;
        notifyObservers();
    }

    private void notifyObservers() {
        observers.forEach(o -> o.onStatusChange(this, status));
    }
}
```

---

## LLD Problems Where Observer Applies

| Problem | Subject | Observers |
|---------|---------|-----------|
| Notification System | Event | EmailHandler, SMSHandler, PushHandler |
| Stock Ticker | Stock | Portfolio, PriceAlert, Dashboard |
| Order Tracking | Order | Customer email, Warehouse, Analytics |
| Splitwise | BalanceSheet | EmailNotifier, PushNotifier |
| Pub/Sub System | Topic | All subscribers |
| Elevator | Floor request | All elevators (pick one to respond) |

---

## Extension Demonstration

This is the cleanest extension example in interviews:

```java
// Adding a push notification channel:
class PushNotifier implements OrderObserver {
    public void onStatusChange(Order order, OrderStatus status) { ... }
}
// Register it:
order.addObserver(new PushNotifier());
// Zero changes to Order, EmailNotifier, SMSNotifier
```

---

## Thread Safety Note

If observers are called from multiple threads, the observer list needs protection:

```java
private final List<OrderObserver> observers = new CopyOnWriteArrayList<>();
// or
private final List<OrderObserver> observers = Collections.synchronizedList(new ArrayList<>());
```

---

## How to Justify in an Interview

> "I'm using Observer here because multiple components need to react to order status changes — email, SMS, analytics. By having Order maintain a list of observers and notify them on change, I decouple the notification channels from the order logic. Adding a push notification channel means adding one new observer class and registering it — zero changes to Order."

---

## Common Mistakes

- Subject holding concrete references to EmailNotifier, SMSNotifier directly (defeats extensibility)
- Synchronous vs asynchronous notification not discussed (heavy observers block the subject)
- Observer list not thread-safe when observers can register/deregister concurrently
- Not providing remove/unsubscribe mechanism (memory leak risk)
