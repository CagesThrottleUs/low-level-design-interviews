# Gap Remediation Guide

Targeted drills for each weakness dimension. Identify your gap from `EVALUATION_RUBRIC.md`, then run the drill here before your next session.

---

## How to Identify Your Gap

After each session, find the 1–2 lowest-scoring dimensions. Match against the sections below. Run the drill 2–3 times before re-testing on a new problem.

---

## Gap 1: Requirements Gathering

**Symptom:** You jump to class diagrams within the first 2 minutes. Interviewer asks "what about X?" and you didn't consider it.

**Root cause:** No structured habit for requirements extraction.

**Drill: Requirements-Only Practice (10 min)**

Take any problem. Spend exactly 10 minutes doing ONLY requirements — no design. Produce:

```
Actors:
- [list every type of person/system that interacts]

Functional Requirements:
- Core use cases (MUST have for MVP)
- Secondary use cases (nice-to-have)

Non-Functional Requirements:
- Concurrency: how many simultaneous users?
- Consistency: is eventual consistency acceptable?
- Persistence: in-memory or durable?
- Scale hints: hundreds vs millions of records?

Assumptions:
- [explicit, not silent]

Out of Scope:
- [what you deliberately will NOT design]
```

Run this drill on 5 different problems without designing anything. Then notice how different your designs become.

**Structured question checklist (memorize this):**
1. Who are the actors?
2. What are the top 3 use cases?
3. Is concurrency a concern?
4. Any external integrations?
5. In-memory OK or need persistence?
6. Any constraints I should know?

---

## Gap 2: Entity Modeling

**Symptom:** Your classes are data bags (only fields + getters/setters). No behavior. Interviewer asks "where does X happen?" and you don't know where to put it.

**Root cause:** Thinking in database tables (rows of data) not objects (data + behavior).

**Drill: Noun-Verb Extraction (15 min)**

Take any problem statement. Run through it:
1. Underline every noun → candidate class
2. Underline every verb → candidate method
3. Assign each verb to the most responsible noun
4. Remove classes that have no behavior (merge into owner class)

**Example — Parking Lot:**
```
Nouns: ParkingLot, Floor, Spot, Vehicle, Ticket, Payment, Gate, Attendant
Verbs: park, unpark, calculate-fee, check-availability, issue-ticket, process-payment

Assignments:
- ParkingLot.findAvailableSpot()
- Spot.canFit(vehicleType), Spot.park(vehicle), Spot.unpark()
- Ticket.calculateFee()
- Payment.process()
- Gate.enter(vehicle), Gate.exit(ticket)
```

**Anti-pattern check:** If a class only has getters/setters, ask: "What does this thing DO?" If nothing, it's a data transfer object (DTO) — don't model it as a domain class.

**Anemic model red flags:**
- `UserManager` class that does everything
- All classes have only `get*/set*` methods
- A single `process()` method that does 200 lines

---

## Gap 3: Design Patterns

**Symptom:** You use long if-else chains for "if payment type is X, do Y". Or you copy-paste logic for each notification type. Or interviewer asks "how would you add a new type?" and you say "add another case to the switch".

**Root cause:** Not recognizing the trigger signals for patterns.

**Drill: Pattern Trigger Recognition (20 min)**

For each of these scenarios, name the pattern and sketch the interface:

| Scenario | Your Pattern | Key Interface |
|----------|-------------|---------------|
| Payment can be CreditCard, UPI, Bitcoin | Strategy | `interface PaymentStrategy { void pay(double amount); }` |
| Order status changes should notify dashboard, email, SMS | Observer | `interface OrderObserver { void onStatusChange(Order o); }` |
| ATM behaves differently when Idle vs HasCard vs PinVerified | State | `interface ATMState { void insertCard(); void enterPin(); void withdraw(); }` |
| Creating different vehicles: Car, Bike, Truck | Factory Method | `abstract Vehicle createVehicle(VehicleType t)` |
| Coffee can have optional milk, sugar, syrup add-ons | Decorator | `abstract class CoffeeDecorator implements Coffee` |
| Building Order with many optional fields | Builder | `OrderBuilder.withItem().withDiscount().build()` |

**Memorize the 4 most common trigger phrases:**
- "multiple types that behave differently" → Strategy
- "multiple listeners/subscribers" → Observer  
- "state-dependent behavior" → State
- "create objects without knowing type upfront" → Factory

**Force-fit warning:** Don't apply a pattern just to show it. Only if it genuinely simplifies the code.

---

## Gap 4: SOLID Principles

**Symptom:** God class with 500 lines. `PaymentProcessor` has a switch on payment type. Interviewer says "where would you add X?" and the answer is "edit this existing class".

**Root cause:** Thinking in procedures, not responsibilities.

**Drill: Violation Hunt (15 min)**

Take your last design solution. Go through each SOLID check:

```
SRP Check: For each class, write one sentence of what it does.
  - If sentence has "and" → SRP violation. Split it.

OCP Check: "To add a new [thing], I must..."
  - If answer is "edit existing class" → OCP violation. Extract interface.

LSP Check: "Can I use [SubClass] everywhere [BaseClass] is used without breaking anything?"
  - If no → LSP violation. Restructure hierarchy.

ISP Check: "Do all implementors of this interface use all its methods?"
  - If no → split the interface.

DIP Check: "Does this class directly instantiate [OtherClass]?"
  - If yes → inject via interface instead.
```

**Most common violation to fix first:** OCP. Change every `if/switch on type` into a Strategy.

```java
// BEFORE (OCP violation)
class PaymentProcessor {
    void process(Payment p) {
        if (p.type == CREDIT_CARD) { ... }
        else if (p.type == UPI) { ... }
        // Adding Bitcoin requires editing this class
    }
}

// AFTER (OCP compliant)
interface PaymentStrategy {
    void pay(double amount);
}
class CreditCardStrategy implements PaymentStrategy { ... }
class UpiStrategy implements PaymentStrategy { ... }
// Adding Bitcoin = new class only, zero changes here
```

---

## Gap 5: Extensibility

**Symptom:** Interviewer says "now add X" and you say "I'd have to refactor the whole thing" or you need to touch 5 classes to add one feature.

**Root cause:** Hardcoded logic instead of abstractions. Same as OCP gap but measured by outcome.

**Drill: Extension Challenge (20 min)**

After completing any design, deliberately try to add these features and count how many existing files you must touch:

1. Add a new payment method
2. Add a new notification channel
3. Add a new vehicle type
4. Add a new pricing strategy (off-peak discount)
5. Add a new user role

**Scoring:** 0 existing files touched = excellent. 1 = acceptable. 2+ = needs refactor.

**Fix pattern:** Every hardcoded `instanceof` or `switch-on-type` is an extensibility smell. Replace with polymorphism.

```java
// BAD — extensibility violation
double calculateFee(Vehicle v) {
    if (v instanceof Car) return hours * 20;
    if (v instanceof Bike) return hours * 10;
    // Adding Truck = edit this method
}

// GOOD — extensible
interface Vehicle {
    double hourlyRate();
}
class Car implements Vehicle { double hourlyRate() { return 20; } }
class Bike implements Vehicle { double hourlyRate() { return 10; } }
// Adding Truck = new class only
```

---

## Gap 6: Concurrency

**Symptom:** You design as if only one user exists. Interviewer asks "what if two users book simultaneously?" and you're caught off-guard.

**Root cause:** Single-threaded mental model.

**Drill: Shared State Audit (10 min)**

After any design, run through every class and answer:
- What mutable state does this class hold?
- Can multiple threads write to it simultaneously?
- What is the worst race condition that could happen?
- What mechanism prevents it?

**Cheat sheet — mechanism by scenario:**

| Shared Resource | Mechanism | Why |
|----------------|-----------|-----|
| Counter (available spots) | `AtomicInteger` | Lock-free increment/decrement |
| Map (spot → ticket) | `ConcurrentHashMap` | Thread-safe map |
| Booking a specific spot | `synchronized` per spot object | Fine-grained, no global lock |
| Account balance | DB transaction + optimistic locking | Atomic read-modify-write |
| Cache | `ReentrantReadWriteLock` | Many readers, exclusive writer |
| Multiple resources | Lock ordering (smallest id first) | Deadlock prevention |

**Add this to every session:** After core design, say "let me think about concurrency." Even if you skip depth, naming it shows awareness.

---

## Gap 7: Communication

**Symptom:** You code in silence. Interviewer asks "why did you choose that?" and you struggle to articulate. Or you narrate what you're doing ("I'm adding a getter") not why.

**Root cause:** No habit of narrating reasoning.

**Drill: Think-Aloud Practice (ongoing)**

Practice sessions: work alone but speak every decision aloud as if interviewer is in room. Record yourself if possible.

**Template for narrating a decision:**
```
"I'm [doing X] because [reason]. The alternative was [Y] but [tradeoff]. 
This approach makes it easy to [extension/test/understand]."
```

**Examples:**
- "I'm extracting a `PricingStrategy` interface because we have hourly, flat-rate, and premium pricing — different algorithms for the same operation. This lets us add new pricing without touching `ParkingTicket`."
- "I'm using composition here, not inheritance, because a `ParkingFloor` HAS spots but isn't a type of spot."
- "I'm using `ConcurrentHashMap` for the spot registry because multiple threads will read frequently and write occasionally — this beats synchronized everywhere."

**What NOT to narrate:** "I'm adding a getter for the name field." Obvious from reading. Narrate reasoning, not mechanics.

---

## Weekly Drill Plan

| Session | Drill | Time |
|---------|-------|------|
| Monday | Requirements-only on 2 new problems | 20 min |
| Tuesday | Noun-verb extraction on same 2 problems | 20 min |
| Wednesday | Pattern trigger drill (table above) | 20 min |
| Thursday | Full 45-min timed mock, score yourself | 45 min |
| Friday | Review lowest dimension, targeted drill | 20 min |

After 4 weeks of this pattern, retest on a problem you did in week 1. Score delta shows growth.
