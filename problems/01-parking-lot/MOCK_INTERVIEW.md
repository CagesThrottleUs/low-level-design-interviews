# Mock Interview Script — Parking Lot

**For the interviewer.** Read this before the session. Do not share with candidate.

---

## Pre-Session Setup

- Set timer: 45 min
- Candidate has blank paper or empty IDE
- You have this script open (hidden from candidate)

---

## Opening (2 min)

Read aloud:

> "Today I'd like you to design a parking lot management system. The system should handle vehicles entering, parking, and exiting. Fees should be calculated on exit. Before you start designing, take a moment to ask any clarifying questions."

Wait for candidate to ask questions. Use this table:

| If candidate asks | Your answer |
|------------------|-------------|
| What vehicle types? | Motorcycle, car, truck — at minimum |
| What spot types? | Compact, large, handicapped. Motorcycles fit anywhere; trucks only in large |
| How is pricing calculated? | Hourly, and rates differ by vehicle type |
| How many floors/spots? | Multiple floors, each with multiple spots — exact numbers don't matter for design |
| Concurrency? | Yes — multiple vehicles can enter/exit simultaneously |
| Persistence needed? | In-memory is fine |
| What if lot is full? | Return an appropriate error/exception |
| Nearest spot assignment? | Yes — assign nearest available spot of appropriate type |
| Anything not listed | "Reasonable assumption — go with what you think is appropriate." |

**If candidate skips requirements and jumps to class diagram immediately:** Note Dimension 1 = 0. Let them proceed.

---

## Phase 1: Requirements Discussion (target: 5–8 min)

Good signal: candidate identifies all 3 vehicle types, 3 spot types, the assignment logic, and pricing. States assumptions explicitly.

Prompt if needed: "Any other actors in this system beyond the driver?"

---

## Phase 2: Core Design (target: 20–25 min)

Let candidate drive. Intervene only if stuck 2+ min.

**Hints — give in order, least specific first:**

If stuck on entities:
1. "What are the main nouns in this problem?"
2. "Think about what a parking spot needs to know and do."
3. "There's a ticket involved — what does it hold?"

If using only data classes (anemic model):
1. "Where does the logic for checking if a spot fits a vehicle live?"
2. "Could the spot know whether it can fit a vehicle type?"

If using instanceof or switch on vehicle type:
1. "If we add electric vehicles, how many places would you need to change?"
2. "Is there a way to make the vehicle type extensible?"

If PricingStrategy not introduced:
1. "If we wanted to add flat-rate pricing alongside hourly — how would your design support that?"

If no thread safety:
— Wait until Phase 4 concurrency probe.

---

## Phase 3: Extension Probe (~35 min mark)

Say:

> "Let's say we want to add electric vehicle spots with charging stations. How would you extend your design?"

Strong answer: Add `ElectricSpot extends ParkingSpot` and `ElectricVehicle extends Vehicle`. PricingStrategy absorbs charging fee. No existing classes changed.

Weak answer: "I'd add another if-else" or "I'd have to change several classes."

---

## Phase 4: Concurrency Probe (~40 min mark)

If candidate never mentioned concurrency:

> "What happens if two cars arrive at the same time and there's only one available spot?"

Strong answer: Identifies race condition on spot assignment. Proposes `synchronized` per spot, or optimistic locking. Notes that a global lock kills throughput — per-spot lock better.

Weak answer: "Oh, that's a good point… I didn't think about that."

---

## Closing (last 2 min)

> "We're almost out of time. If you had 30 more minutes, what would you improve?"

Good answers: "I'd add proper error handling for the full-lot case," "I'd think more carefully about the spot selection algorithm," "I'd revisit the concurrency model."

---

## Scoring Checklist

| Dimension | Evidence to look for | Score (0–3) |
|-----------|---------------------|-------------|
| Requirements | Asked about vehicle types, spot types, pricing, concurrency | |
| Entity Modeling | ParkingSpot with behavior (canFit, park, unpark), Vehicle hierarchy, Ticket, PricingStrategy | |
| Design Patterns | PricingStrategy (Strategy pattern), ideally Factory for spot creation | |
| SOLID | No god class, OCP on vehicle types and pricing | |
| Extensibility | EV extension absorbed cleanly | |
| Concurrency | Race condition on spot assignment identified + solution | |
| Communication | Narrated WHY at each decision point | |
| **TOTAL** | | **/21** |
