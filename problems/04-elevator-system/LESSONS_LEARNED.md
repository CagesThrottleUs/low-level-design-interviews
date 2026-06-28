# Lessons Learned — Elevator System

---

## Mistake 1: Forgetting Door State

**What happens:** Elevator modeled as just moving up/down/idle. No door open state. No invariant preventing movement while doors are open.

**Why it's wrong:** Real elevators won't move with open doors. Without this state, the design is incorrect and fails LSP — any simulation using this design would produce wrong behavior.

**Fix:**
```java
// Elevator states: IDLE, MOVING_UP, MOVING_DOWN, DOORS_OPEN
// Invariant: moveUp/moveDown throw in DoorsOpenState
class DoorsOpenState implements ElevatorState {
    public void moveUp(Elevator e) {
        throw new IllegalStateException("Cannot move — doors are open");
    }
}
```

**Gap dimension:** Entity Modeling (Dimension 2)

---

## Mistake 2: No Separation of External vs Internal Requests

**What happens:** Candidate treats a floor button press and a cabin button press identically in the same queue.

**Why it's wrong:** They're handled differently. External requests need elevator dispatch (which elevator to send?). Internal requests go directly to the elevator's queue. Conflating them makes the controller logic messy.

**Fix:**
```java
abstract class Request { int targetFloor; }
class ExternalRequest extends Request {
    Direction direction; // UP or DOWN
    // Needs controller to select an elevator
}
class InternalRequest extends Request {
    // Goes directly to elevator's queue
}
```

**Gap dimension:** Entity Modeling (Dimension 2)

---

## Mistake 3: Hardcoded Scheduling in Controller

**What happens:** `ElevatorController.dispatch()` has the scheduling logic directly — "find nearest idle elevator" — hardcoded.

**Why it's wrong:** Adding SCAN scheduling requires modifying `ElevatorController` — OCP violation.

**Fix:**
```java
interface SchedulingStrategy {
    Elevator selectElevator(List<Elevator> elevators, ExternalRequest req);
}
// Controller injected with strategy
class ElevatorController {
    private final SchedulingStrategy scheduler;
    // New algorithm = new class, controller untouched
}
```

**Gap dimension:** Design Patterns + SOLID (Dimensions 3, 4)

---

## Mistake 4: Using Boolean Flags for Direction and Door State

**What happens:** `boolean isMovingUp, isMovingDown, isDoorsOpen` — same flag-soup as Vending Machine.

**Why it's wrong:** Combinations of flags create 8 logical states, many of which are invalid (isMovingUp AND isDoorsOpen = true). Hard to prevent invalid combinations.

**Fix:** State pattern. Each state class only allows the transitions valid in that state.

**Gap dimension:** Design Patterns (Dimension 3)

---

## Problem-Specific Gotchas

- SCAN: elevator sweeps all pending stops in current direction before reversing — like a disk head
- Priority queue for stops: order by floor (ascending when going up, descending going down) not FIFO
- Multiple elevators: controller selects "best" elevator per request — worth designing even if algorithm is simple
- Idle elevator closest to requested floor is a good default scheduling heuristic

---

## Self-Assessment Checklist

- [ ] Does elevator have a DoorsOpen state?
- [ ] Does DoorsOpen state prevent movement?
- [ ] Are external requests (floor button) separated from internal requests (cabin button)?
- [ ] Is scheduling logic behind an interface (not hardcoded in controller)?
- [ ] Is request queue thread-safe (PriorityBlockingQueue or synchronized)?
- [ ] Can I add SCAN scheduling by adding a new class only?
- [ ] Is direction an enum (UP/DOWN/IDLE), not a boolean?

---

## Remediation Targets

| Dimension | If you scored 0–1 | Go to |
|-----------|------------------|-------|
| Entity Modeling | Missing door state, flag soup | GAP_REMEDIATION.md#gap-2 — noun-verb extraction |
| Design Patterns | No State on elevator | GAP_REMEDIATION.md#gap-3 — state trigger: "behavior differs per mode" |
| SOLID | Scheduling hardcoded in controller | GAP_REMEDIATION.md#gap-4 — OCP violation hunt |
| Concurrency | Unsafe request queue | GAP_REMEDIATION.md#gap-6 — shared state audit |
