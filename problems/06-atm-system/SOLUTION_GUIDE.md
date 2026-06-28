# Solution Guide — ATM System

**Read after your attempt.**

---

## Entity Map

| Class/Interface | Type | Responsibility |
|----------------|------|---------------|
| `ATM` | Concrete | Context; holds state, card reader, cash dispenser; delegates ops to current state |
| `ATMState` | Interface | `insertCard()`, `enterPin()`, `selectTransaction()`, `executeTransaction()`, `ejectCard()` |
| `IdleState` | Concrete | Only `insertCard()` is valid |
| `CardInsertedState` | Concrete | `enterPin()`; tracks attempts; locks on 3rd failure; `ejectCard()` returns card |
| `PinVerifiedState` | Concrete | `selectTransaction()` and `ejectCard()` valid |
| `TransactionState` | Concrete | `executeTransaction()` valid; returns to PinVerified on completion |
| `Card` | Concrete | Card number, account ID |
| `Transaction` | Interface | `execute(account, cashDispenser): TransactionResult` |
| `WithdrawTransaction` | Concrete | Check balance + check ATM cash; debit + dispense atomically |
| `BalanceInquiry` | Concrete | Read-only balance query |
| `DepositTransaction` | Concrete | Credit account + increment ATM cash |
| `BankSystem` | Concrete (singleton) | Holds accounts; validates PIN; authorizes transactions |
| `Account` | Concrete | Balance (in cents), PIN hash, locked status |
| `CashDispenser` | Concrete | Available denominations + amounts; `dispense(amount)` |

---

## State Machine

```
[IDLE] ──insertCard──> [CARD_INSERTED]
[CARD_INSERTED] ──enterPin(correct)──> [PIN_VERIFIED]
[CARD_INSERTED] ──enterPin(wrong ×3)──> [IDLE] + card locked
[CARD_INSERTED] ──ejectCard──> [IDLE]
[PIN_VERIFIED] ──selectTransaction──> [TRANSACTION]
[PIN_VERIFIED] ──ejectCard──> [IDLE]
[TRANSACTION] ──executeTransaction──> [PIN_VERIFIED] (loop for more transactions)
[TRANSACTION] ──ejectCard──> [IDLE]
```

---

## Design Patterns

### State — ATM Lifecycle

```java
interface ATMState {
    void insertCard(ATM atm, Card card);
    void enterPin(ATM atm, String pin);
    void selectTransaction(ATM atm, Transaction transaction);
    void executeTransaction(ATM atm);
    void ejectCard(ATM atm);
}

class CardInsertedState implements ATMState {
    private int pinAttempts = 0;

    public void enterPin(ATM atm, String pin) {
        Account account = atm.getBankSystem().getAccount(atm.getCurrentCard().getAccountId());
        if (account.validatePin(pin)) {
            pinAttempts = 0;
            atm.setState(atm.getPinVerifiedState());
        } else {
            pinAttempts++;
            if (pinAttempts >= 3) {
                account.lock();
                System.out.println("Card locked after 3 failed attempts.");
                ejectCard(atm);
            } else {
                System.out.println("Incorrect PIN. " + (3 - pinAttempts) + " attempts remaining.");
            }
        }
    }

    public void ejectCard(ATM atm) {
        atm.returnCard();
        atm.setState(atm.getIdleState());
    }

    public void insertCard(ATM atm, Card card) { throw new IllegalStateException("Card already inserted"); }
    public void selectTransaction(ATM atm, Transaction t) { throw new IllegalStateException("Enter PIN first"); }
    public void executeTransaction(ATM atm) { throw new IllegalStateException("Select transaction first"); }
}
```

### Strategy / Polymorphism — Transaction Types

```java
interface Transaction {
    TransactionResult execute(Account account, CashDispenser dispenser);
}

class WithdrawTransaction implements Transaction {
    private final long amountCents;

    public TransactionResult execute(Account account, CashDispenser dispenser) {
        if (account.getBalanceCents() < amountCents) {
            return TransactionResult.failure("Insufficient funds");
        }
        if (!dispenser.canDispense(amountCents)) {
            return TransactionResult.failure("ATM has insufficient cash");
        }
        // Atomic: both must succeed or neither
        boolean debited = account.debit(amountCents);
        if (!debited) return TransactionResult.failure("Debit failed");
        dispenser.dispense(amountCents);
        return TransactionResult.success("Dispensed $" + (amountCents / 100));
    }
}
```

---

## Atomicity: The Key Design Challenge

Withdrawal must be atomic — account debited AND cash dispensed, or neither.

**Problem:** Two sequential operations that can each fail independently.

**Approach 1 — Debit first, dispense second:**
```java
boolean debited = account.debit(amount); // can fail: insufficient funds
if (!debited) return failure;
boolean dispensed = dispenser.dispense(amount); // can fail: no cash in ATM
if (!dispensed) {
    account.credit(amount); // compensating transaction — roll back debit
    return failure;
}
```

**Approach 2 — Reserve first:**
```java
// Check both conditions atomically before doing either
if (account.getBalance() < amount) return failure;
if (!dispenser.canDispense(amount)) return failure;
// Both checks passed — now execute
account.debit(amount);
dispenser.dispense(amount);
```

**Interview answer:** "I'd check both preconditions before executing either operation. For true atomicity in production, this would use a database transaction or saga pattern."

---

## Concurrency: Concurrent Withdrawals

**Scenario:** Same account at two ATMs, both attempting to withdraw $500 from $700 balance.

**Race:** Both read balance $700, both decide to proceed, both debit $500 → balance goes to -$300.

**Fix:** `account.debit()` must be synchronized:

```java
class Account {
    private long balanceCents;

    public synchronized boolean debit(long amount) {
        if (balanceCents < amount) return false;
        balanceCents -= amount;
        return true;
    }
}
```

Or use optimistic locking: check-and-set with compare-and-exchange on balance.

---

## CashDispenser

```java
class CashDispenser {
    private final Map<Integer, Integer> denominations = new TreeMap<>(Collections.reverseOrder());
    // denomination → count; e.g., {100: 10, 50: 20, 20: 50}

    public boolean canDispense(long amountCents) {
        long dollars = amountCents / 100;
        return canMakeChange(new HashMap<>(denominations), dollars);
    }

    public void dispense(long amountCents) {
        long dollars = amountCents / 100;
        for (Map.Entry<Integer, Integer> entry : denominations.entrySet()) {
            int denomination = entry.getKey();
            int needed = (int)(dollars / denomination);
            int available = entry.getValue();
            int use = Math.min(needed, available);
            denominations.put(denomination, available - use);
            dollars -= use * denomination;
        }
    }
}
```

---

## Extension Points

| "Now add..." | Change |
|-------------|--------|
| New transaction type | New class implementing `Transaction` |
| PIN change | `ChangePinTransaction implements Transaction` |
| Transfer funds | `TransferTransaction` — debit source, credit target; discuss compensation on failure |
| Maintenance mode | New ATM state; all customer ops rejected |

---

## What Strong Candidates Do Differently

- PIN attempt counter lives in `CardInsertedState` (not in ATM context) — it resets when state changes
- Withdrawal checks both preconditions BEFORE executing either operation
- `Account.debit()` is synchronized — the ATM doesn't manage locking, the account does
- `CashDispenser` uses greedy denomination selection (largest first)
- Explicitly states: "In production, debit + dispense would be inside a distributed transaction or compensating transaction"

## What Average Candidates Miss

- Flag soup instead of State pattern
- No atomicity discussion for withdraw
- PIN attempts tracked globally (not reset after state change)
- No CashDispenser entity — dispense logic in state or ATM directly
- `BankSystem` as a god object that also manages ATM state
