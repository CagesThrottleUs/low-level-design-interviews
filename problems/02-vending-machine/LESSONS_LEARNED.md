# Lessons Learned — Vending Machine

---

## Mistake 1: Boolean Flag Soup Instead of State Pattern

**What happens:** Candidate uses `boolean hasInsertedMoney`, `boolean productSelected`, `boolean isDispensing` to track machine state. Logic scattered across giant if-else blocks.

**Why it's wrong:** Adding a new state (e.g., "maintenance mode") requires modifying every conditional block. New combinations of flags create invalid states that are hard to prevent. Violates OCP.

**Fix:**
```java
// Before — flag soup
void insertCoin(Coin coin) {
    if (!hasInsertedMoney && !isDispensing) {
        balance += coin.getValue();
        hasInsertedMoney = true;
    } else if (hasInsertedMoney && !isDispensing) {
        balance += coin.getValue();
    } else {
        System.out.println("Invalid operation");
    }
}

// After — State pattern
// IdleState.insertCoin() handles the first case
// HasMoneyState.insertCoin() handles the second
// DispensingState.insertCoin() throws invalid operation
// Each state only knows what's valid for IT
```

**Gap dimension:** Design Patterns (Dimension 3)

---

## Mistake 2: Not Returning Money on Out-of-Stock

**What happens:** Candidate checks stock before accepting payment (good), but doesn't think about the race: item was in stock when selected but sold out by the time dispense happens.

**Why it's wrong:** Customer loses money.

**Fix:** Check stock atomically during dispense, not just during selection:
```java
// In DispensingState.dispense():
if (!machine.getInventory().decrement(productCode)) {
    // Stock ran out — return money
    machine.returnBalance();
    machine.setState(machine.getIdleState());
    return;
}
```

**Gap dimension:** Concurrency + Correctness (Dimensions 6, 2)

---

## Mistake 3: Recreating State Objects on Every Transition

**What happens:** Each transition creates `new IdleState()`, `new HasMoneyState()` etc.

**Why it's wrong:** Wasteful allocations for objects that are stateless singletons (their state is on the machine context, not themselves).

**Fix:** Create all states once in `VendingMachine` constructor, reuse on transitions:
```java
class VendingMachine {
    private final VendingMachineState idleState = new IdleState(this);
    private final VendingMachineState hasMoneyState = new HasMoneyState(this);
    // ...
    void setState(VendingMachineState s) { this.currentState = s; }
}
// State transitions: machine.setState(machine.getIdleState())
```

**Gap dimension:** Entity Modeling (Dimension 2)

---

## Mistake 4: VendingMachine Does Everything (God Class)

**What happens:** VendingMachine has methods for all state-dependent operations directly, with big if-else blocks.

**Fix:** All behavior stays in state classes. VendingMachine only:
- Holds current state
- Delegates operations to current state
- Exposes mutators for state classes to update context

**Gap dimension:** SOLID (Dimension 4)

---

## Problem-Specific Gotchas

- Change calculation: `balance - price`, returned in coins — tricky if you try to implement exact change (optional scope)
- Transition back to `HasMoneyState` (not Idle) if payment is insufficient after product selection
- `Coin` as enum with values is cleaner than passing raw integers
- The machine holds state objects as final fields — they don't change, only `currentState` pointer changes

---

## Self-Assessment Checklist

- [ ] Did I use the State pattern (not boolean flags)?
- [ ] Are invalid operations rejected in each state (not in one giant if-else)?
- [ ] Does cancel in any state return the customer's money?
- [ ] Is out-of-stock handled at both selection AND dispense time?
- [ ] Are state objects reused (not recreated on every transition)?
- [ ] Is inventory decrement atomic or synchronized?
- [ ] Can I add "maintenance mode" by adding a new state class only?

---

## Remediation Targets

| Dimension | If you scored 0–1 | Go to |
|-----------|------------------|-------|
| Design Patterns | Used boolean flags, no State | GAP_REMEDIATION.md#gap-3 — state trigger: "behavior depends on current state" |
| Entity Modeling | VendingMachine does all logic | GAP_REMEDIATION.md#gap-2 — noun-verb extraction |
| SOLID | OCP violation on state transitions | GAP_REMEDIATION.md#gap-4 — violation hunt |
| Concurrency | Race on inventory decrement | GAP_REMEDIATION.md#gap-6 — shared state audit |
