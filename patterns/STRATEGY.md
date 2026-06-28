# Strategy Pattern

**Category:** Behavioral
**LLD interview frequency:** Tier 1 — must know cold

---

## Intent

Define a family of algorithms, encapsulate each one, and make them interchangeable. Lets the algorithm vary independently from clients that use it.

**One sentence:** Replace `if/switch on type` with a pluggable algorithm behind an interface.

---

## Trigger Signals

Apply Strategy when you hear:
- "Multiple types that behave differently" (payment methods, pricing models, sort orders)
- "Adding a new X should not require changing existing code"
- "Switch statement that grows every time a new type is added"
- "Configurability at runtime" (user can choose algorithm)

---

## Structure

```java
// 1. Interface defines the algorithm contract
interface PricingStrategy {
    double calculateFee(ParkingTicket ticket);
}

// 2. Concrete strategies implement it
class HourlyPricingStrategy implements PricingStrategy {
    public double calculateFee(ParkingTicket ticket) {
        long hours = getHours(ticket);
        return hours * rateFor(ticket.getVehicle().getType());
    }
}

class FlatRatePricingStrategy implements PricingStrategy {
    public double calculateFee(ParkingTicket ticket) { return 50.0; }
}

// 3. Context holds the strategy (injected, not hardcoded)
class ExitGate {
    private final PricingStrategy pricingStrategy;

    public ExitGate(PricingStrategy strategy) {
        this.pricingStrategy = strategy;
    }

    public Payment processExit(ParkingTicket ticket) {
        double fee = pricingStrategy.calculateFee(ticket);
        return new Payment(fee);
    }
}
```

---

## LLD Problems Where Strategy Applies

| Problem | Strategy for |
|---------|-------------|
| Parking Lot | Pricing (hourly, flat, surge) |
| Splitwise | Expense splitting (equal, percentage, exact) |
| Elevator | Scheduling algorithm (nearest car, SCAN) |
| Notification System | Delivery channel (email, SMS, push) |
| Rate Limiter | Algorithm (token bucket, sliding window, fixed window) |
| Ride Share | Driver matching (nearest, rating-based) |
| E-commerce | Discount/coupon calculation |
| Sorting | Sort order (relevance, price, rating) |

---

## vs. Hardcoded Switch

```java
// BAD — switch on type; adding "CRYPTO" payment requires editing this method
double calculateFee(VehicleType type, long hours) {
    return switch (type) {
        case CAR -> hours * 10.0;
        case MOTORCYCLE -> hours * 5.0;
        case TRUCK -> hours * 20.0;
    };
}

// GOOD — new vehicle type = new class, zero changes to this method
interface VehiclePricing { double hourlyRate(); }
class CarPricing implements VehiclePricing { public double hourlyRate() { return 10.0; } }
```

---

## How to Justify in an Interview

> "I'm using Strategy for pricing because we have multiple algorithms (hourly vs flat rate) and the requirement hints at adding more. By abstracting behind PricingStrategy, adding surge pricing means adding one new class with zero changes to ExitGate. The algorithm varies independently of the client."

---

## Common Mistakes

- Force-fitting Strategy when there's only one algorithm (YAGNI)
- Using Strategy but still passing the type as a parameter (defeats the purpose)
- Not injecting the strategy (instantiating inside the context = defeats extensibility)
