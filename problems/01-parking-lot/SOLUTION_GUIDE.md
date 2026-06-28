# Solution Guide — Parking Lot

**Read after your attempt.**

---

## Entity Map

| Class/Interface | Type | Responsibility |
|----------------|------|---------------|
| `ParkingLot` | Concrete (singleton) | Holds floors; coordinates entry/exit |
| `ParkingFloor` | Concrete | Holds spots; finds available spot |
| `ParkingSpot` | Abstract | Knows type, occupancy, vehicle; park/unpark |
| `CompactSpot` | Concrete extends ParkingSpot | Accepts motorcycles and cars |
| `LargeSpot` | Concrete extends ParkingSpot | Accepts all vehicle types |
| `HandicappedSpot` | Concrete extends ParkingSpot | Accepts motorcycles and cars (special access) |
| `Vehicle` | Abstract | Has licensePlate, vehicleType |
| `Car` / `Motorcycle` / `Truck` | Concrete extends Vehicle | Define their SpotType requirement |
| `ParkingTicket` | Concrete | Entry time, spot, vehicle — issued on entry |
| `PricingStrategy` | Interface | `calculateFee(ticket): double` |
| `HourlyPricingStrategy` | Concrete | Fee = hours * hourlyRate[vehicleType] |
| `FlatRatePricingStrategy` | Concrete | Fixed fee regardless of duration |
| `Payment` | Concrete | Holds fee, status, method |
| `EntryGate` | Concrete | Finds spot, issues ticket |
| `ExitGate` | Concrete | Calculates fee, processes payment, frees spot |

---

## Class Diagram (text)

```
ParkingLot ──has-many──> ParkingFloor
ParkingFloor ──has-many──> ParkingSpot (abstract)
    ParkingSpot
        ├── CompactSpot
        ├── LargeSpot
        └── HandicappedSpot

Vehicle (abstract)
    ├── Car
    ├── Motorcycle
    └── Truck

PricingStrategy (interface)
    ├── HourlyPricingStrategy
    └── FlatRatePricingStrategy

ParkingTicket ──references──> ParkingSpot
ParkingTicket ──references──> Vehicle
ExitGate ──uses──> PricingStrategy
```

---

## Design Patterns Applied

### Pattern 1: Strategy — Pricing

**Where:** `PricingStrategy` interface, consumed by `ExitGate`
**Why:** Pricing logic varies (hourly vs flat rate) and we'll add more strategies (surge pricing, subscription). Strategy isolates the varying algorithm behind a stable interface.
**Benefit:** Add new pricing = new class. Zero changes to ExitGate or Ticket.

```java
interface PricingStrategy {
    double calculateFee(ParkingTicket ticket);
}

class HourlyPricingStrategy implements PricingStrategy {
    private static final Map<VehicleType, Double> RATES = Map.of(
        VehicleType.MOTORCYCLE, 5.0,
        VehicleType.CAR, 10.0,
        VehicleType.TRUCK, 20.0
    );

    public double calculateFee(ParkingTicket ticket) {
        long hours = ChronoUnit.HOURS.between(ticket.getEntryTime(), Instant.now());
        hours = Math.max(1, hours); // minimum 1 hour
        return hours * RATES.get(ticket.getVehicle().getType());
    }
}
```

### Pattern 2: Template / Polymorphism — Spot Fitting

**Where:** `ParkingSpot.canFit(VehicleType)` — each subclass defines what it accepts
**Why:** Avoids `instanceof` checks in allocation logic. The spot knows what it can hold.
**Benefit:** Add ElectricSpot = new class with its own `canFit` logic.

```java
abstract class ParkingSpot {
    private boolean occupied;
    private Vehicle parkedVehicle;
    private final String id;

    public abstract boolean canFit(VehicleType vehicleType);

    public synchronized boolean park(Vehicle v) {
        if (occupied || !canFit(v.getType())) return false;
        this.parkedVehicle = v;
        this.occupied = true;
        return true;
    }

    public synchronized Vehicle unpark() {
        Vehicle v = this.parkedVehicle;
        this.parkedVehicle = null;
        this.occupied = false;
        return v;
    }
}

class CompactSpot extends ParkingSpot {
    public boolean canFit(VehicleType type) {
        return type == VehicleType.MOTORCYCLE || type == VehicleType.CAR;
    }
}

class LargeSpot extends ParkingSpot {
    public boolean canFit(VehicleType type) {
        return true; // all vehicle types
    }
}
```

---

## Core Algorithm: Spot Allocation

```java
class ParkingFloor {
    private final List<ParkingSpot> spots;

    public Optional<ParkingSpot> findAvailableSpot(VehicleType vehicleType) {
        return spots.stream()
            .filter(s -> !s.isOccupied() && s.canFit(vehicleType))
            .findFirst(); // "first" = nearest in ordered list
    }
}

class EntryGate {
    private final ParkingLot lot;

    public ParkingTicket enter(Vehicle vehicle) {
        ParkingSpot spot = lot.getFloors().stream()
            .map(floor -> floor.findAvailableSpot(vehicle.getType()))
            .filter(Optional::isPresent)
            .map(Optional::get)
            .findFirst()
            .orElseThrow(() -> new LotFullException("No available spot for " + vehicle.getType()));

        boolean parked = spot.park(vehicle);
        if (!parked) throw new ConcurrentBookingException("Spot taken by concurrent request");

        return new ParkingTicket(vehicle, spot, Instant.now());
    }
}
```

---

## Concurrency Handling

**Shared mutable state:**
- `ParkingSpot.occupied` + `ParkingSpot.parkedVehicle` — race condition if two threads call `park()` simultaneously on same spot

**Solution:** Synchronize `park()` at the spot level (not global lock):

```java
// In ParkingSpot:
public synchronized boolean park(Vehicle v) {
    if (occupied) return false; // double-check after acquiring lock
    this.parkedVehicle = v;
    this.occupied = true;
    return true;
}
```

**Why per-spot, not global:** A global lock on `ParkingLot` means Car A entering floor 1 blocks Car B entering floor 3. Per-spot synchronization only blocks on the same spot — better throughput.

**Pattern in EntryGate:** Find candidate spot (read-only, concurrent OK), then attempt `park()` (synchronized). If `park()` returns false, retry — find next available spot.

---

## Extension Points

| "Now add..." | Change required | Pattern enabling it |
|-------------|----------------|---------------------|
| Flat-rate pricing | Add `FlatRatePricingStrategy` | Strategy |
| Surge pricing (peak hours) | Add `SurgePricingStrategy` | Strategy |
| Electric vehicle spots | Add `ElectricSpot extends ParkingSpot` + `ElectricVehicle extends Vehicle` | Polymorphism |
| Monthly subscriber pricing | Add `SubscriptionPricingStrategy` | Strategy |
| New vehicle type (Bus) | Add `Bus extends Vehicle` + override `canFit` responses | Polymorphism |

---

## Full Code Sketch

```java
enum VehicleType { MOTORCYCLE, CAR, TRUCK }
enum SpotType { COMPACT, LARGE, HANDICAPPED }

abstract class Vehicle {
    protected String licensePlate;
    protected VehicleType type;
    public VehicleType getType() { return type; }
}

class Car extends Vehicle {
    public Car(String plate) {
        this.licensePlate = plate;
        this.type = VehicleType.CAR;
    }
}

// ... Motorcycle, Truck similar

abstract class ParkingSpot {
    protected String id;
    protected boolean occupied;
    protected Vehicle parkedVehicle;

    public abstract boolean canFit(VehicleType type);

    public synchronized boolean park(Vehicle v) {
        if (occupied || !canFit(v.getType())) return false;
        parkedVehicle = v; occupied = true; return true;
    }

    public synchronized Vehicle unpark() {
        Vehicle v = parkedVehicle; parkedVehicle = null; occupied = false; return v;
    }

    public boolean isOccupied() { return occupied; }
}

class LargeSpot extends ParkingSpot {
    public boolean canFit(VehicleType type) { return true; }
}

class CompactSpot extends ParkingSpot {
    public boolean canFit(VehicleType type) {
        return type == VehicleType.MOTORCYCLE || type == VehicleType.CAR;
    }
}

class ParkingTicket {
    private final Vehicle vehicle;
    private final ParkingSpot spot;
    private final Instant entryTime;
    // getters...
}

interface PricingStrategy {
    double calculateFee(ParkingTicket ticket);
}

class HourlyPricingStrategy implements PricingStrategy {
    public double calculateFee(ParkingTicket ticket) {
        long hours = Math.max(1, ChronoUnit.HOURS.between(
            ticket.getEntryTime(), Instant.now()));
        return hours * rateFor(ticket.getVehicle().getType());
    }
    private double rateFor(VehicleType t) {
        return switch (t) {
            case MOTORCYCLE -> 5.0;
            case CAR -> 10.0;
            case TRUCK -> 20.0;
        };
    }
}

class ParkingFloor {
    private final int floorNumber;
    private final List<ParkingSpot> spots;

    public Optional<ParkingSpot> findAvailableSpot(VehicleType type) {
        return spots.stream()
            .filter(s -> !s.isOccupied() && s.canFit(type))
            .findFirst();
    }
}

class ParkingLot {
    private static ParkingLot instance;
    private final List<ParkingFloor> floors;

    public static synchronized ParkingLot getInstance() {
        if (instance == null) instance = new ParkingLot();
        return instance;
    }

    public Optional<ParkingSpot> findSpot(VehicleType type) {
        return floors.stream()
            .map(f -> f.findAvailableSpot(type))
            .filter(Optional::isPresent)
            .map(Optional::get)
            .findFirst();
    }
}
```

---

## What Strong Candidates Do Differently

- Put `canFit()` on `ParkingSpot`, not in a utility or if-else block
- Use `synchronized` on individual spot's `park()`, not a global lock
- Extract `PricingStrategy` without being prompted
- Explicitly state "I'm using composition here, not a big switch"
- Discuss the retry loop in `EntryGate` when a spot becomes taken between find and park

## What Average Candidates Miss

- Anemic `ParkingSpot` with only getters — no `park()`, `unpark()`, `canFit()`
- Global lock on `ParkingLot.park()` — kills concurrency
- Pricing logic embedded in `ParkingTicket` or `ExitGate` with switch statement
- No `Vehicle` hierarchy — all vehicle logic as if-else on a type field
- Singleton for `ParkingLot` used without thread-safe initialization
