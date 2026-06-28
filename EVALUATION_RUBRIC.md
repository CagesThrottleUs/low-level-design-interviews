# Evaluation Rubric — LLD Interviews

7-dimension scoring system. Use for every practice session.

## How to Score

Score each dimension 0–3. Total max = 21. Note specific evidence for each score.

---

## Dimension 1: Requirements Gathering (0–3)

**What it measures:** Did candidate define the problem correctly before designing?

| Score | Evidence |
|-------|----------|
| 0 | Jumped to design with no requirements discussion |
| 1 | Listed some features, missed key actors or constraints |
| 2 | Identified actors, functional reqs, made explicit assumptions |
| 3 | All of 2 + identified non-functional reqs (concurrency, scale hints), negotiated scope for the time box |

**Questions good candidates ask:**
- Who are the actors? (customer, admin, system)
- What are the core use cases vs nice-to-haves?
- Any concurrency requirements? Multiple users simultaneously?
- What scale are we designing for? (affects data structure choices)
- Any external systems to integrate?
- Read-heavy or write-heavy?

**Anti-patterns:**
- Asking too many trivial questions (stalling)
- Not asking any questions (jumping to code)
- Designing for every edge case without time-boxing

---

## Dimension 2: Entity Modeling (0–3)

**What it measures:** Are the right classes identified with the right responsibilities?

| Score | Evidence |
|-------|----------|
| 0 | Wrong classes, or one god class for everything |
| 1 | Right entities identified but as anemic data bags (only fields, no behavior) |
| 2 | Right entities with meaningful methods, relationships roughly correct |
| 3 | All of 2 + correct relationship types (composition vs inheritance), no SRP violations, behavior in the right class |

**Key checks:**
- Are nouns → classes, verbs → methods?
- Does each class have a clear single responsibility?
- Are there god classes (Manager, Controller, Handler doing too much)?
- Is behavior in the domain model or leaked to service layer?
- Is inheritance used only for true IS-A relationships?
- Are enums used for status/type fields (not magic strings)?

**Anti-pattern example:**
```
// BAD — anemic model
class ParkingSpot {
    private SpotType type;
    private boolean occupied;
    // only getters/setters — no behavior
}

// GOOD — rich model
class ParkingSpot {
    private SpotType type;
    private boolean occupied;
    private Vehicle parkedVehicle;

    public boolean canFit(VehicleType vehicleType) { ... }
    public void park(Vehicle v) { ... }
    public Vehicle unpark() { ... }
}
```

---

## Dimension 3: Design Patterns (0–3)

**What it measures:** Are patterns applied correctly and with justification?

| Score | Evidence |
|-------|----------|
| 0 | No patterns applied, rigid if-else chains |
| 1 | Pattern named but not actually implemented |
| 2 | Pattern implemented but no justification given |
| 3 | Pattern implemented + reason stated ("I used Strategy here because payment methods vary and we'll add more — this lets us add without modifying PaymentProcessor") |

**Pattern trigger signals — recognize these:**

| Signal in requirements | Pattern to apply |
|------------------------|-----------------|
| "Multiple types of X that behave differently" | Strategy |
| "Multiple components need to know when X changes" | Observer |
| "Object behavior depends on its current state" | State |
| "Need to create objects without knowing exact type" | Factory Method |
| "Add features to object without changing its class" | Decorator |
| "Complex object with many optional parameters" | Builder |
| "Need to undo/redo operations" | Command |

**Force-fitting warning:** Never apply a pattern just to show you know it. Justify or skip.

---

## Dimension 4: SOLID Compliance (0–3)

**What it measures:** Does the design follow SOLID principles (implicitly, not as a checklist)?

| Score | Evidence |
|-------|----------|
| 0 | Multiple violations — god classes, concrete dependencies, rigid if-else |
| 1 | 1–2 clear violations |
| 2 | Minor violations only, overall structure is clean |
| 3 | Clean — each class has one reason to change, abstractions correct, dependencies inverted |

**Quick violation checklist:**

- [ ] **SRP violated:** Class has more than one reason to change (e.g., UserService sends email + persists data)
- [ ] **OCP violated:** Adding new payment method requires editing PaymentProcessor (not just adding)
- [ ] **LSP violated:** Subclass breaks behavior of base class (Square-Rectangle problem)
- [ ] **ISP violated:** Interface has methods some implementors don't use (fat interface)
- [ ] **DIP violated:** High-level class directly instantiates low-level class (no interface)

---

## Dimension 5: Extensibility (0–3)

**What it measures:** Can the design absorb a new requirement cleanly?

| Score | Evidence |
|-------|----------|
| 0 | Adding any new requirement requires rewriting core logic |
| 1 | New requirement possible but touches multiple existing classes |
| 2 | New requirement can be added by adding a new class + minimal changes |
| 3 | New requirement = new class only, zero existing code modified |

**Extension probe questions** (interviewers ask these):
- "Now add a new payment method — credit card, UPI, crypto"
- "Now add a new vehicle type — motorcycle, electric car"
- "Now add a discount strategy for premium users"
- "Now add a new notification channel — push, SMS, email"

**Ideal answer:** "I can add `CryptoPaymentStrategy implements PaymentStrategy` with no changes to `PaymentProcessor`."

---

## Dimension 6: Concurrency Awareness (0–3)

**What it measures:** Does candidate proactively identify and address shared mutable state?

| Score | Evidence |
|-------|----------|
| 0 | Concurrency never mentioned, design has obvious race conditions |
| 1 | Noticed when prompted, surface-level answer |
| 2 | Proactively identified shared state, named the solution |
| 3 | Full treatment: identified race conditions, chose appropriate mechanism (lock granularity, atomic, concurrent collections), mentioned deadlock risk if applicable |

**Standard probe:** "What happens if two users book the same seat simultaneously?"

**Good answer elements:**
- Identify the race condition: "Both could read `available=true`, both proceed to book"
- Name the mechanism: optimistic locking, pessimistic lock, synchronized block, `AtomicBoolean`, database transaction
- Address granularity: "I'd use per-spot lock not a global lock to maximize throughput"
- Mention deadlock if multiple locks involved

**Common mechanisms by scenario:**

| Scenario | Mechanism |
|----------|-----------|
| Counter increment | `AtomicInteger` |
| Spot/seat availability | Optimistic locking + DB transaction OR `synchronized` per spot |
| Shared map | `ConcurrentHashMap` |
| Multiple resources, ordering matters | Lock ordering to prevent deadlock |
| Cache reads | `ReentrantReadWriteLock` (multiple readers, exclusive writer) |

---

## Dimension 7: Communication (0–3)

**What it measures:** Does candidate narrate decisions so interviewer can engage?

| Score | Evidence |
|-------|----------|
| 0 | Silent throughout, interviewer can't follow reasoning |
| 1 | Answers questions when asked, doesn't volunteer |
| 2 | Narrates major decisions proactively |
| 3 | Real-time narration of reasoning, explicitly states tradeoffs, invites feedback ("does this approach look right to you?") |

**Good narration examples:**
- "I'm choosing composition over inheritance here because the relationship is HAS-A not IS-A"
- "I'm using Strategy for payment because we have multiple types and will likely add more"
- "I could use Singleton for the parking lot manager but I'll avoid it — hides dependencies"
- "This is a tradeoff: per-spot locking gives better throughput but more complexity"

---

## Scoring Summary

```
Score after session:

Requirements:     _/3
Entity Modeling:  _/3
Design Patterns:  _/3
SOLID:            _/3
Extensibility:    _/3
Concurrency:      _/3
Communication:    _/3

TOTAL:            _/21
```

**Benchmarks:**
- 0–9: Significant gaps. Focus on fundamentals (entity modeling + requirements)
- 10–14: Developing. Address specific failing dimensions
- 15–17: Interview-ready. Sharpen weakest 1–2 dimensions
- 18–21: Strong. Practice speed and complex problems

## Post-Session Protocol

1. Mark scores per dimension
2. Identify the 1–2 lowest dimensions
3. Go to `GAP_REMEDIATION.md`, find that dimension's targeted drill
4. Do drill before next session
5. Log in `progress/TRACKER.md`
