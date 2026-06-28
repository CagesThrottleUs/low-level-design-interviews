# Mock Interview Script — Elevator System

**For the interviewer.**

---

## Opening

> "Design an elevator control system for a multi-story building with multiple elevators. Before designing, ask any clarifying questions."

| If candidate asks | Answer |
|------------------|--------|
| Number of floors/elevators? | N floors, M elevators — configurable; exact number doesn't matter for design |
| Scheduling algorithm? | Start with any working algorithm; optimal is a bonus |
| Direction buttons on floors? | Yes — up and down buttons on each floor |
| Internal destination buttons? | Yes — inside each elevator |
| Capacity? | Yes — max capacity per elevator, but enforcement is secondary |
| Emergency features? | Out of scope for now |
| Concurrency? | Yes — multiple requests can arrive simultaneously |

---

## Phase 2: Core Design

**Key signals to watch:**

1. Does candidate model elevator as a state machine (Idle/Moving/DoorsOpen)?
2. Does candidate separate floor requests (external) from destination requests (internal)?
3. Does candidate use a scheduling strategy?

If candidate models elevator as one flat class with flags:
> "What states can an elevator be in? How does its behavior differ per state?"

If candidate uses one big `handleRequest()` method with if-else:
> "If we wanted to swap to a different scheduling algorithm, how easy would that be in your design?"

If candidate forgets door state:
> "What ensures the elevator doesn't move while doors are open?"

---

## Extension Probe (~35 min)

> "Currently you have a basic scheduling algorithm. How would you swap it for SCAN scheduling — where the elevator doesn't reverse direction until it reaches the top or bottom?"

Strong: `SchedulingStrategy` interface already in place — add `SCANStrategy implements SchedulingStrategy`. Zero changes to `ElevatorController`.

Weak: "I'd add more conditions to the existing dispatch logic."

---

## Concurrency Probe (~40 min)

> "Two people on different floors press the button at the same time. What are the concurrent risks?"

Strong: Race on elevator's request queue. Mentions `PriorityBlockingQueue` or `synchronized` queue. Also: elevator state transitions should be atomic.

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | External + internal requests, door state, multi-elevator, scheduling | |
| Entity Modeling | Elevator as state machine, Controller, Request types, Floor | |
| Design Patterns | State pattern on Elevator, Strategy on scheduling | |
| SOLID | SchedulingStrategy as interface (OCP), single-responsibility per class | |
| Extensibility | New scheduling algorithm = new class, zero controller changes | |
| Concurrency | Queue safety, state transition atomicity | |
| Communication | Explained FSM for elevator, WHY scheduling strategy abstracted | |
| **TOTAL** | | **/21** |
