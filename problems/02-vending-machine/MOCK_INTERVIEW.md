# Mock Interview Script — Vending Machine

**For the interviewer.** Read this before the session.

---

## Opening

> "Design a vending machine. It should accept payment, allow product selection, dispense the item, and return change. Ask any clarifying questions before you start."

| If candidate asks | Answer |
|------------------|--------|
| Payment types? | Coins and bills; skip card for MVP |
| Multiple coins? | Yes — cumulative until sufficient |
| Out of stock? | Machine should inform customer and return their money |
| Cancel? | Yes — customer can cancel anytime before dispensing; returns full amount |
| Concurrency? | One customer at a time per machine |
| How many products? | Multiple slots (A1–D5 style); each slot has name, price, quantity |

---

## Phase 2: Core Design — What to Watch For

**Key signal:** Does candidate naturally go toward a State pattern?

Good candidates will say something like: "The machine behaves differently depending on what state it's in — idle vs has-money vs dispensing. I'll model each state as a class."

Average candidates will use a series of if-else or boolean flags:
```java
if (moneyInserted && productSelected && !dispensing) { ... }
```

If they go the if-else route and don't self-correct by 15 min, give hint:

> "How many different 'modes' does the machine have? How does each mode affect what operations are valid?"

---

## Extension Probe (~35 min)

> "Now add support for card payments (Stripe-style). How would your design change?"

Strong: Extracted `PaymentStrategy` or `PaymentHandler` interface means adding card = new class, zero changes to `VendingMachine`. Or if payment isn't extracted, candidate identifies the gap and states how to fix it.

---

## Concurrency Probe (~40 min, if not raised)

> "What if two people try to buy the last item simultaneously?"

Strong: identifies the race on inventory decrement. Proposes `synchronized` on dispense operation or atomic check-and-decrement.

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | Payment types, cancellation, out-of-stock, cumulative payment | |
| Entity Modeling | State objects with behavior (not just booleans), Inventory as entity | |
| Design Patterns | State pattern (core FSM), ideally Strategy for payment | |
| SOLID | No god VendingMachine class; each state handles its own logic | |
| Extensibility | New payment type = new class, no changes to machine | |
| Concurrency | Race on inventory decrement identified | |
| Communication | Explained WHY State pattern over if-else flags | |
| **TOTAL** | | **/21** |
