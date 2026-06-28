# Lessons Learned — Parking Lot

Common mistakes candidates make on this problem. Check yourself after each attempt.

---

## Mistake 1: Anemic ParkingSpot

**What happens:** Candidate creates `ParkingSpot` with only `type`, `isOccupied`, `vehicleId` fields and getters/setters. All logic for checking fit and parking goes into a `ParkingManager` or `ParkingFloor` as if-else blocks.

**Why it's wrong:** Domain logic scattered outside the domain object. Adding a new spot type requires hunting down every if-else and updating it.

**Fix:**
```java
// Before — anemic
class ParkingSpot {
    SpotType type;
    boolean occupied;
    // just getters/setters
}
// Logic leaks to:
if (spot.getType() == COMPACT && vehicle.getType() == MOTORCYCLE) { ... }

// After — rich domain model
abstract class ParkingSpot {
    public abstract boolean canFit(VehicleType type);
    public synchronized boolean park(Vehicle v) { ... }
    public synchronized Vehicle unpark() { ... }
}
class CompactSpot extends ParkingSpot {
    public boolean canFit(VehicleType t) { return t == MOTORCYCLE || t == CAR; }
}
```

**Gap dimension:** Entity Modeling (Dimension 2)

---

## Mistake 2: Global Lock on ParkingLot

**What happens:** Candidate wraps the entire `ParkingLot.park()` or `findAndPark()` method with `synchronized`. Works correctly but kills throughput.

**Why it's wrong:** Car on floor 1 blocks car on floor 5 from parking. Real parking lots handle these independently.

**Fix:** Lock at the spot level, not the lot level.

```java
// Before — coarse lock
public synchronized ParkingTicket park(Vehicle v) {
    ParkingSpot spot = findSpot(v.getType());
    spot.setOccupied(true);
    return new ParkingTicket(v, spot);
}

// After — fine-grained lock
// ParkingSpot.park() is synchronized
// EntryGate retries if spot taken between find and park:
public ParkingTicket enter(Vehicle v) {
    while (true) {
        Optional<ParkingSpot> spot = lot.findSpot(v.getType());
        if (spot.isEmpty()) throw new LotFullException();
        if (spot.get().park(v)) { // synchronized inside
            return new ParkingTicket(v, spot.get(), Instant.now());
        }
        // spot was taken by concurrent thread — retry
    }
}
```

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 3: Pricing Logic in ParkingTicket or ExitGate (OCP Violation)

**What happens:** Candidate puts fee calculation inside `ParkingTicket.calculateFee()` or `ExitGate.processExit()` with a switch on vehicle type.

**Why it's wrong:** Adding flat-rate pricing or surge pricing requires modifying an existing class.

**Fix:**
```java
// Before — OCP violation
double calculateFee(ParkingTicket ticket) {
    long hours = getHours(ticket);
    return switch (ticket.getVehicleType()) {
        case CAR -> hours * 10;
        case TRUCK -> hours * 20;
        // Adding surge pricing = edit this method
    };
}

// After — Strategy pattern
interface PricingStrategy {
    double calculateFee(ParkingTicket ticket);
}
// ExitGate gets PricingStrategy via constructor injection
// New pricing = new class, zero changes to ExitGate
```

**Gap dimension:** Design Patterns (Dimension 3) + SOLID (Dimension 4)

---

## Mistake 4: No Vehicle Hierarchy

**What happens:** Candidate uses a single `Vehicle` class with a `VehicleType` enum field. All vehicle-type-specific logic is if-else blocks on the enum value.

**Why it's wrong:** Adding a new vehicle type (Bus, Electric) requires hunting every if-else. Violates OCP.

**Fix:**
```java
// Before
class Vehicle {
    VehicleType type; // enum
    // no behavior
}
// Everywhere: if (v.type == CAR) { ... }

// After
abstract class Vehicle {
    public abstract VehicleType getType();
    public abstract double getHourlyRate();
}
class Car extends Vehicle {
    public VehicleType getType() { return VehicleType.CAR; }
    public double getHourlyRate() { return 10.0; }
}
```

**Gap dimension:** SOLID / Extensibility (Dimensions 4, 5)

---

## Mistake 5: Ignoring the Race Condition Between Find and Park

**What happens:** Candidate synchronizes `park()` but doesn't handle the case where spot becomes taken between `findSpot()` and `park()`.

**Why it's wrong:** Two threads can both `findSpot()` and get the same spot. One parks, the other calls `park()` — without checking the return value, double-booking occurs.

**Fix:** Check `park()` return value. Retry if it returns false.

```java
// Must handle the false case
boolean parked = spot.park(vehicle);
if (!parked) {
    // spot taken by concurrent request — find next spot or throw
}
```

**Gap dimension:** Concurrency (Dimension 6)

---

## Problem-Specific Gotchas

- Motorcycle fits in any spot type — missing this makes allocation logic wrong
- Truck only fits in LARGE — candidates often forget this constraint
- The "nearest spot" requirement means spots should be ordered/indexed, not just a set
- Minimum 1-hour billing is common real-world practice — worth mentioning even if not implemented
- `ParkingLot` as singleton is fine, but use thread-safe initialization: `double-checked locking` or enum singleton

---

## Self-Assessment Checklist

After your attempt, verify:

- [ ] Does `ParkingSpot` have `park()`, `unpark()`, and `canFit()` methods (not just fields)?
- [ ] Is `PricingStrategy` an interface with at least one concrete implementation?
- [ ] Is the Vehicle hierarchy used (not just a `VehicleType` enum with if-else)?
- [ ] Is concurrency addressed at the spot level (not global lock)?
- [ ] Does `EntryGate` retry if `park()` returns false due to concurrent booking?
- [ ] Are spot types represented as subclasses (not a switch in `canFit`)?
- [ ] Is there a `ParkingTicket` with entry time that feeds into fee calculation?

---

## Remediation Targets

| Dimension | If you scored 0–1 | Go to |
|-----------|------------------|-------|
| Entity Modeling | Anemic ParkingSpot | GAP_REMEDIATION.md#gap-2 + redo noun-verb extraction on this problem |
| Design Patterns | No PricingStrategy | GAP_REMEDIATION.md#gap-3 + pattern trigger drill |
| SOLID | Switch on VehicleType | GAP_REMEDIATION.md#gap-4 + violation hunt |
| Extensibility | Adding EV requires editing 3+ classes | GAP_REMEDIATION.md#gap-5 + extension challenge |
| Concurrency | No per-spot sync, no retry loop | GAP_REMEDIATION.md#gap-6 + shared state audit |
