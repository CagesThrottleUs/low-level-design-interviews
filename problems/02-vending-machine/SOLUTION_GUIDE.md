# Solution Guide — Vending Machine

**Read after your attempt.**

---

## Entity Map

| Class/Interface | Type | Responsibility |
|----------------|------|---------------|
| `VendingMachine` | Concrete | Holds current state + inventory + balance; delegates all ops to state |
| `VendingMachineState` | Interface | `insertCoin()`, `selectProduct()`, `dispense()`, `cancel()` |
| `IdleState` | Concrete | Only allows coin insertion |
| `HasMoneyState` | Concrete | Allows more coins, product selection, cancel |
| `ProductSelectedState` | Concrete | Validates payment, transitions to dispensing |
| `DispensingState` | Concrete | Dispenses item, returns change, resets to idle |
| `Inventory` | Concrete | Map of slot code → Product; thread-safe stock management |
| `Product` | Concrete | Name, price, quantity |
| `Coin` | Enum | PENNY(1), NICKEL(5), DIME(10), QUARTER(25) — value in cents |

---

## State Machine Diagram

```
[IDLE] ──insertCoin──> [HAS_MONEY]
[HAS_MONEY] ──insertCoin──> [HAS_MONEY]  (cumulative)
[HAS_MONEY] ──selectProduct──> [PRODUCT_SELECTED]
[HAS_MONEY] ──cancel──> [IDLE]  (return money)
[PRODUCT_SELECTED] ──dispense──> [DISPENSING]
[PRODUCT_SELECTED] ──(insufficient funds)──> [HAS_MONEY]
[DISPENSING] ──(complete)──> [IDLE]
```

---

## Design Pattern: State

**Core insight:** The machine has 4 distinct behavioral modes. Instead of:
```java
if (hasInsertedMoney && !productSelected) { ... }
else if (hasInsertedMoney && productSelected && !dispensing) { ... }
```

Model each mode as a class:
```java
interface VendingMachineState {
    void insertCoin(Coin coin);
    void selectProduct(String productCode);
    void dispense();
    void cancel();
}
```

Each state class handles only the operations valid in that state, and throws/ignores invalid ones:

```java
class IdleState implements VendingMachineState {
    private final VendingMachine machine;

    public void insertCoin(Coin coin) {
        machine.addBalance(coin.getValue());
        System.out.println("Inserted: " + coin + ". Balance: " + machine.getBalance());
        machine.setState(machine.getHasMoneyState());
    }

    public void selectProduct(String code) {
        throw new IllegalStateException("Insert money first");
    }

    public void dispense() {
        throw new IllegalStateException("No product selected");
    }

    public void cancel() {
        System.out.println("Nothing to cancel");
    }
}
```

```java
class HasMoneyState implements VendingMachineState {
    private final VendingMachine machine;

    public void insertCoin(Coin coin) {
        machine.addBalance(coin.getValue());
    }

    public void selectProduct(String code) {
        Product p = machine.getInventory().get(code);
        if (p == null || p.getQuantity() == 0) {
            System.out.println("Out of stock. Returning money.");
            machine.returnBalance();
            machine.setState(machine.getIdleState());
            return;
        }
        machine.setSelectedProduct(p);
        machine.setState(machine.getProductSelectedState());
    }

    public void cancel() {
        machine.returnBalance();
        machine.setState(machine.getIdleState());
    }

    public void dispense() {
        throw new IllegalStateException("Select a product first");
    }
}
```

```java
class ProductSelectedState implements VendingMachineState {
    private final VendingMachine machine;

    public void dispense() {
        Product p = machine.getSelectedProduct();
        if (machine.getBalance() < p.getPrice()) {
            System.out.println("Insufficient funds. Need " + (p.getPrice() - machine.getBalance()) + " more.");
            machine.setState(machine.getHasMoneyState());
            return;
        }
        machine.setState(machine.getDispensingState());
        machine.getCurrentState().dispense(); // chain to dispensing
    }

    public void insertCoin(Coin coin) { machine.addBalance(coin.getValue()); }
    public void cancel() { machine.returnBalance(); machine.setState(machine.getIdleState()); }
    public void selectProduct(String c) { throw new IllegalStateException("Already selected"); }
}
```

```java
class DispensingState implements VendingMachineState {
    private final VendingMachine machine;

    public void dispense() {
        Product p = machine.getSelectedProduct();
        machine.getInventory().decrement(p);
        int change = machine.getBalance() - p.getPrice();
        System.out.println("Dispensing: " + p.getName());
        if (change > 0) System.out.println("Change: " + change);
        machine.resetBalance();
        machine.setSelectedProduct(null);
        machine.setState(machine.getIdleState());
    }

    public void insertCoin(Coin c) { throw new IllegalStateException("Dispensing in progress"); }
    public void selectProduct(String c) { throw new IllegalStateException("Dispensing in progress"); }
    public void cancel() { throw new IllegalStateException("Cannot cancel during dispensing"); }
}
```

---

## VendingMachine Context

```java
class VendingMachine {
    private VendingMachineState currentState;
    private final VendingMachineState idleState;
    private final VendingMachineState hasMoneyState;
    private final VendingMachineState productSelectedState;
    private final VendingMachineState dispensingState;

    private final Inventory inventory;
    private int balance; // in cents
    private Product selectedProduct;

    public VendingMachine(Inventory inventory) {
        this.inventory = inventory;
        idleState = new IdleState(this);
        hasMoneyState = new HasMoneyState(this);
        productSelectedState = new ProductSelectedState(this);
        dispensingState = new DispensingState(this);
        currentState = idleState;
    }

    public void insertCoin(Coin coin) { currentState.insertCoin(coin); }
    public void selectProduct(String code) { currentState.selectProduct(code); }
    public void dispense() { currentState.dispense(); }
    public void cancel() { currentState.cancel(); }

    // package-private for state classes
    void setState(VendingMachineState state) { this.currentState = state; }
    void addBalance(int amount) { balance += amount; }
    void returnBalance() { System.out.println("Returning: " + balance); balance = 0; }
    void resetBalance() { balance = 0; }
    // ... getters for states, inventory, balance, selectedProduct
}
```

---

## Concurrency Handling

Race condition: two customers at the same machine (unlikely in practice but interviewers ask).

**Shared state:** `inventory.decrement()` — concurrent decrement on last item.

```java
class Inventory {
    private final Map<String, Product> slots = new ConcurrentHashMap<>();

    public synchronized boolean decrement(String code) {
        Product p = slots.get(code);
        if (p == null || p.getQuantity() <= 0) return false;
        p.setQuantity(p.getQuantity() - 1);
        return true;
    }
}
// Or use AtomicInteger for quantity field
```

---

## Extension: Payment Strategy

```java
interface PaymentStrategy {
    boolean processPayment(double amount);
}
class CoinPayment implements PaymentStrategy { ... }
class CardPayment implements PaymentStrategy { ... }
```

`VendingMachine` gets `PaymentStrategy` injected — card support = new class, zero machine changes.

---

## What Strong Candidates Do Differently

- Immediately identify FSM and explain State pattern before coding
- Pre-create all state instances in `VendingMachine` constructor (avoid recreation)
- State objects hold a back-reference to the machine context to mutate balance/inventory
- Handle out-of-stock BEFORE taking payment (or return money if discovered after)
- Explicitly state: "Each state handles only what's valid — invalid ops throw or are ignored"

## What Average Candidates Miss

- Flag soup: `boolean hasInsertedMoney, boolean productSelected, boolean isDispensing`
- Duplicate logic across state-like conditions
- Not returning money on out-of-stock discovery
- Not considering the cancel flow
- Global state mutation with no thread safety on inventory
