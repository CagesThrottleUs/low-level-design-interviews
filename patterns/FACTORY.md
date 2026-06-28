# Factory / Factory Method Pattern

**Category:** Creational
**LLD interview frequency:** Tier 2 — high value

---

## Intent

Define an interface for creating an object, but let subclasses or a factory method decide which class to instantiate.

**One sentence:** Decouple object creation from usage — caller doesn't know or care which concrete class it gets.

---

## Trigger Signals

Apply Factory when you hear:
- "Create objects without knowing the exact type at compile time"
- "Need to switch implementations based on configuration or input"
- "Object creation is complex and should be centralized"
- Problems: Parking Lot (spot/vehicle types), Notification (channel creation), Payment (processor creation)

---

## Two Variants

### 1. Static Factory Method (most common in interviews)

```java
class VehicleFactory {
    public static Vehicle create(VehicleType type, String licensePlate) {
        return switch (type) {
            case CAR -> new Car(licensePlate);
            case MOTORCYCLE -> new Motorcycle(licensePlate);
            case TRUCK -> new Truck(licensePlate);
        };
    }
}

// Usage — caller doesn't know which Vehicle subclass
Vehicle vehicle = VehicleFactory.create(VehicleType.CAR, "ABC-123");
```

### 2. Factory Method (subclass decides)

```java
abstract class NotificationSender {
    // Factory method — subclass provides the channel
    protected abstract NotificationChannel createChannel();

    public void send(String message) {
        NotificationChannel channel = createChannel(); // polymorphic creation
        channel.deliver(message);
    }
}

class EmailNotificationSender extends NotificationSender {
    protected NotificationChannel createChannel() { return new EmailChannel(); }
}

class SMSNotificationSender extends NotificationSender {
    protected NotificationChannel createChannel() { return new SMSChannel(); }
}
```

---

## LLD Problems Where Factory Applies

| Problem | Factory for |
|---------|------------|
| Parking Lot | Vehicle creation, ParkingSpot creation |
| Notification System | NotificationChannel creation by type |
| Payment | PaymentProcessor creation by method |
| Chess | Piece creation by type |
| Vending Machine | Product creation from slot code |

---

## How to Justify in an Interview

> "I'm using a Factory here to centralize vehicle creation. Callers just pass a VehicleType — they don't know whether they're getting a Car, Motorcycle, or Truck. If we add a new vehicle type, we update the factory only, not every place that creates vehicles."

---

## Common Mistakes

- Using Factory where simple constructor is fine (over-engineering)
- Factory that returns a base type but callers immediately cast to concrete type (defeats purpose)
- Factory method that's just a long switch — this is fine initially; pair with interface when adding types
