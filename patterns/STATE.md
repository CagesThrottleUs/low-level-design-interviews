# State Pattern

**Category:** Behavioral
**LLD interview frequency:** Tier 1 — must know cold

---

## Intent

Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.

**One sentence:** Model each "mode" of an object as its own class; the object delegates all operations to its current mode.

---

## Trigger Signals

Apply State when you hear:
- "The system behaves differently depending on its current state"
- "Certain operations are only valid in certain modes"
- "Object goes through a defined lifecycle" (states + transitions)
- Problems: ATM, Vending Machine, Elevator, Order Lifecycle, Traffic Light

---

## Structure

```java
// 1. State interface — defines all operations an object can perform
interface VendingMachineState {
    void insertCoin(Coin coin);
    void selectProduct(String code);
    void dispense();
    void cancel();
}

// 2. Concrete state classes — implement only what's valid in that state
class IdleState implements VendingMachineState {
    private final VendingMachine machine;

    public void insertCoin(Coin coin) {
        machine.addBalance(coin.getValue());
        machine.setState(machine.getHasMoneyState()); // transition
    }

    public void selectProduct(String code) {
        throw new IllegalStateException("Insert money first");
    }

    public void dispense() { throw new IllegalStateException("No product selected"); }
    public void cancel() { /* nothing to do */ }
}

// 3. Context — delegates to current state
class VendingMachine {
    private VendingMachineState currentState;
    // All state objects pre-created and reused
    private final VendingMachineState idleState = new IdleState(this);
    private final VendingMachineState hasMoneyState = new HasMoneyState(this);

    public void insertCoin(Coin coin) { currentState.insertCoin(coin); }
    public void selectProduct(String code) { currentState.selectProduct(code); }

    void setState(VendingMachineState state) { this.currentState = state; }
    VendingMachineState getIdleState() { return idleState; }
    VendingMachineState getHasMoneyState() { return hasMoneyState; }
}
```

---

## LLD Problems Where State Applies

| Problem | States |
|---------|--------|
| Vending Machine | Idle, HasMoney, ProductSelected, Dispensing |
| ATM | Idle, CardInserted, PinVerified, TransactionInProgress |
| Elevator | Idle, MovingUp, MovingDown, DoorsOpen |
| Order Lifecycle | Created, Confirmed, Shipped, Delivered, Cancelled |
| Traffic Light | Red, Yellow, Green |
| Document Workflow | Draft, Review, Approved, Published |
| Chess / Game Turn | Player1Turn, Player2Turn, GameOver |

---

## State Machine Diagram First

Before coding, always draw the FSM:
```
[IDLE] ──insertCoin──> [HAS_MONEY]
[HAS_MONEY] ──selectProduct──> [PRODUCT_SELECTED]
[HAS_MONEY] ──cancel──> [IDLE]
[PRODUCT_SELECTED] ──dispense──> [DISPENSING]
[DISPENSING] ──done──> [IDLE]
```

Then map each node to a class.

---

## vs. Boolean Flags

```java
// BAD — flag soup; 2 booleans = 4 states, some invalid
boolean hasMoney;
boolean productSelected;

void insertCoin(Coin c) {
    if (!hasMoney && !productSelected) { hasMoney = true; ... }
    else if (hasMoney && !productSelected) { ... }
    // state combinations explode
}

// GOOD — State pattern; each class handles only what's valid for it
class IdleState { void insertCoin(Coin c) { /* only valid op here */ } }
class HasMoneyState { void insertCoin(Coin c) { /* add to balance */ } }
// Adding "MaintenanceState" = one new class, zero changes to others
```

---

## How to Justify in an Interview

> "The vending machine has four distinct behavioral modes. Instead of tracking mode with booleans and if-else chains, I'll use the State pattern — each mode becomes a class implementing the same interface. The machine delegates every operation to its current state, and invalid operations throw or are ignored within the state class. Adding a maintenance mode means adding one new class with zero changes to existing states."

---

## Common Mistakes

- Recreating state objects on every transition (should be pre-created in context)
- State objects accessing context's private fields directly (use package-private or context methods)
- Trying to implement State as an enum with switch — misses the OCP benefit
- Forgetting that the context holds state objects as fields (not recreated per call)
