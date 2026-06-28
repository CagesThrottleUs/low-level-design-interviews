# Solution Guide — Elevator System

**Read after your attempt.**

---

## Entity Map

| Class/Interface | Type | Responsibility |
|----------------|------|---------------|
| `ElevatorController` | Concrete | Receives requests; dispatches using SchedulingStrategy |
| `Elevator` | Concrete | State machine; processes request queue; moves floor-by-floor |
| `ElevatorState` | Interface | `openDoors()`, `closeDoors()`, `moveUp()`, `moveDown()` |
| `IdleState` | Concrete | Accepts requests; opens doors if at requested floor |
| `MovingUpState` | Concrete | Moving up; stops at requested floors in path |
| `MovingDownState` | Concrete | Moving down; stops at requested floors in path |
| `DoorsOpenState` | Concrete | Doors open; closes before any movement |
| `SchedulingStrategy` | Interface | `selectElevator(List<Elevator>, Request): Elevator` |
| `NearestCarStrategy` | Concrete | Pick nearest idle elevator |
| `SCANStrategy` | Concrete | SCAN disk scheduling algorithm |
| `Request` | Abstract | Has target floor; ExternalRequest adds direction |
| `ExternalRequest` | Concrete | Floor button press — has floor + direction |
| `InternalRequest` | Concrete | Cabin button press — has destination floor |

---

## State Machine: Elevator States

```
[IDLE] ──requestArrives──> [MOVING_UP or MOVING_DOWN]
[MOVING_UP] ──reachTargetFloor──> [DOORS_OPEN]
[MOVING_DOWN] ──reachTargetFloor──> [DOORS_OPEN]
[DOORS_OPEN] ──timeout/close──> [IDLE or MOVING_UP or MOVING_DOWN]
```

Door invariant: transition from DOORS_OPEN to MOVING only after doors fully closed.

---

## Design Patterns

### Strategy — Scheduling Algorithm

```java
interface SchedulingStrategy {
    Elevator selectElevator(List<Elevator> elevators, ExternalRequest request);
}

class NearestCarStrategy implements SchedulingStrategy {
    public Elevator selectElevator(List<Elevator> elevators, ExternalRequest req) {
        return elevators.stream()
            .filter(e -> e.getState() instanceof IdleState)
            .min(Comparator.comparingInt(e -> Math.abs(e.getCurrentFloor() - req.getFloor())))
            .orElse(elevators.get(0)); // fallback: any elevator
    }
}
```

### State — Elevator FSM

```java
interface ElevatorState {
    void openDoors(Elevator elevator);
    void closeDoors(Elevator elevator);
    void moveUp(Elevator elevator);
    void moveDown(Elevator elevator);
}

class IdleState implements ElevatorState {
    public void moveUp(Elevator e) {
        System.out.println("Elevator " + e.getId() + " moving up from floor " + e.getCurrentFloor());
        e.setState(new MovingUpState());
    }
    public void moveDown(Elevator e) { e.setState(new MovingDownState()); }
    public void openDoors(Elevator e) { e.setState(new DoorsOpenState()); }
    public void closeDoors(Elevator e) { /* already closed */ }
}

class MovingUpState implements ElevatorState {
    public void moveUp(Elevator e) { e.setCurrentFloor(e.getCurrentFloor() + 1); }
    public void openDoors(Elevator e) {
        // arrived at a floor — open doors
        e.setState(new DoorsOpenState());
        e.getCurrentState().openDoors(e);
    }
    public void moveDown(Elevator e) { throw new IllegalStateException("Can't reverse while moving up"); }
    public void closeDoors(Elevator e) { /* doors already closed */ }
}

class DoorsOpenState implements ElevatorState {
    public void closeDoors(Elevator e) { e.setState(new IdleState()); }
    public void moveUp(Elevator e) { throw new IllegalStateException("Close doors before moving"); }
    public void moveDown(Elevator e) { throw new IllegalStateException("Close doors before moving"); }
    public void openDoors(Elevator e) { /* already open */ }
}
```

---

## Core Request Processing

```java
class Elevator {
    private final int id;
    private int currentFloor;
    private ElevatorState state;
    private final PriorityBlockingQueue<Integer> requestQueue; // floors to stop at

    public synchronized void addRequest(int targetFloor) {
        requestQueue.add(targetFloor);
    }

    public void processNextRequest() {
        if (requestQueue.isEmpty()) return;
        int targetFloor = requestQueue.peek();

        if (currentFloor < targetFloor) {
            state.moveUp(this);
            currentFloor++;
        } else if (currentFloor > targetFloor) {
            state.moveDown(this);
            currentFloor--;
        } else {
            // arrived
            requestQueue.poll();
            state.openDoors(this);
            // simulate door open delay
            state.closeDoors(this);
        }
    }
}
```

---

## Concurrency

**Shared state:** `requestQueue` — multiple requests arrive concurrently.
**Solution:** `PriorityBlockingQueue` (thread-safe) + `synchronized` on `addRequest`.

**Elevator state transitions:** Wrap state changes in `synchronized` blocks to prevent invalid transitions from concurrent threads.

---

## Extension Points

| "Now add..." | Change |
|-------------|--------|
| SCAN scheduling | Add `SCANStrategy implements SchedulingStrategy` |
| Weight enforcement | Add `currentWeight`, `maxCapacity` to Elevator; check in DoorsOpenState |
| Emergency mode | Add `EmergencyState`; all other transitions refused |
| Maintenance mode | Add `MaintenanceState` |

---

## What Strong Candidates Do Differently

- Separate ExternalRequest (floor button) from InternalRequest (cabin button) — different routing
- Door invariant stated explicitly: no movement from DoorsOpen state
- `PriorityQueue` ordered by proximity to current floor (SCAN-lite)
- Strategy for scheduling — immediately explains extensibility

## What Average Candidates Miss

- Forget door state entirely — elevator can teleport between floors
- One class for all logic with boolean flags for direction and door status
- No separation of external vs internal requests — same queue, same handling
- No scheduling strategy abstraction — hardcoded "nearest" logic in controller
