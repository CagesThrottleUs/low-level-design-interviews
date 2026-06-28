# Elevator System

**Difficulty:** Foundation-Intermediate
**Category:** State Machine + Scheduling Algorithm
**Time Box:** 45 min (discussion) / 90 min (machine coding)
**Key Patterns:** State (elevator states), Strategy (scheduling algorithm), Observer (floor requests)
**Asked at:** Google, Amazon, Uber, Microsoft, Lyft

---

## Problem Statement

Design an elevator control system for a building with multiple elevators and multiple floors. The system receives requests from people pressing buttons on floors (external requests) and from people inside elevators pressing destination floor buttons (internal requests). The system must dispatch the most appropriate elevator and schedule stops efficiently.

---

## Actors

| Actor | Description |
|-------|-------------|
| Floor Passenger | Presses up/down button on a floor to call elevator |
| Elevator Passenger | Presses destination floor button inside elevator |
| Elevator Controller | Receives all requests; dispatches elevators |
| Elevator | Moves between floors; opens doors; handles internal requests |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: Passengers on a floor can request an elevator (with direction: up or down)
- FR2: Passengers inside an elevator can request a destination floor
- FR3: Controller assigns an elevator to each floor request
- FR4: Elevator moves to requested floors in efficient order
- FR5: Elevator opens doors on arrival, closes before moving

**Secondary (implement if time allows):**
- FR6: Multiple elevators managed by single controller
- FR7: SCAN scheduling (elevator doesn't reverse until top/bottom — like disk scheduling)
- FR8: Weight capacity enforcement
- FR9: Emergency stop and maintenance mode

---

## Non-Functional Requirements

- **Concurrency:** Multiple requests arrive simultaneously
- **Correctness:** Elevator should never move with doors open
- **Efficiency:** Minimize total wait time (scheduling algorithm matters)

---

## Constraints and Assumptions

- Building has N floors, M elevators (configurable at init)
- Elevators can only move one floor at a time (continuous movement)
- Doors must be fully closed before movement
- An elevator has a maximum capacity (weight or person count)
- Out of scope: real-time hardware control, fire emergency mode (unless time allows)

---

## Good Clarifying Questions to Ask

1. How many elevators? How many floors?
2. Is scheduling algorithm important — or just "any working assignment"?
3. Should one elevator handle multiple stops in sequence, or just one stop at a time?
4. Do floor requests have direction (up/down) or just a floor number?
5. Is capacity enforcement required?
6. What's the failure mode if an elevator breaks mid-journey?

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **ElevatorController** — receives requests; dispatches elevators using a scheduling algorithm
- **Elevator** — state machine (Idle/Moving/DoorsOpen); processes request queue; moves floor by floor
- **ElevatorState** — interface: Idle, MovingUp, MovingDown, DoorsOpen
- **Request** — external (floor + direction) or internal (destination floor); both go into a queue
- **SchedulingStrategy** — interface; FirstComeFirstServed, SCAN, etc. implement it
- **Floor** — has an up button and down button; triggers external requests
- **Direction** — enum: UP, DOWN

</details>
