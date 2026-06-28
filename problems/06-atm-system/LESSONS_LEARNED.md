# Lessons Learned — ATM System

---

## Mistake 1: Flag Soup Instead of State Pattern

**What happens:** `boolean isCardInserted, isPinVerified, isTransactionActive` tracking ATM mode. All logic in one giant `processInput()` with nested if-else.

**Why it's wrong:** Adding "maintenance mode" requires touching every condition. Invalid state combinations (isPinVerified = true but isCardInserted = false) are possible.

**Fix:** State pattern — `IdleState`, `CardInsertedState`, `PinVerifiedState`, `TransactionState`. Each state rejects invalid operations with an exception or message.

**Gap dimension:** Design Patterns (Dimension 3)

---

## Mistake 2: No Withdrawal Atomicity Discussion

**What happens:** Candidate writes `account.debit(amount)` then `dispenser.dispense(amount)` as two separate steps with no failure handling between them.

**Why it's wrong:** If `dispenser.dispense()` fails after `account.debit()` succeeds, money is lost — customer is debited but receives no cash.

**Fix:** Check BOTH preconditions before executing EITHER operation:
```java
// Check all conditions first
if (account.getBalance() < amount) return failure("Insufficient funds");
if (!dispenser.canDispense(amount)) return failure("ATM has no cash");
// Only now execute both
account.debit(amount);
dispenser.dispense(amount);
// On production: wrap in distributed transaction or saga pattern
```

**Gap dimension:** Concurrency / Correctness (Dimensions 6, 2)

---

## Mistake 3: PIN Attempt Counter on the ATM (Not the State)

**What happens:** `atm.pinAttempts` field tracks failed attempts. On card eject, it's not reset. Next user inherits previous user's failed attempts.

**Why it's wrong:** Counter belongs to the `CardInsertedState` instance, not the ATM. Each card insertion creates a fresh state (or resets the state's counter).

**Fix:**
```java
class CardInsertedState implements ATMState {
    private int pinAttempts = 0; // belongs to THIS state instance, not the ATM
    // Resets automatically because a new state instance is created per card insertion
}
```

**Gap dimension:** Entity Modeling (Dimension 2)

---

## Mistake 4: No Account Locking on 3 Failed PINs

**What happens:** ATM returns card after 3 failures but doesn't tell the bank to lock the account. Customer can insert card again at same or different ATM.

**Fix:**
```java
// CardInsertedState.enterPin():
if (++pinAttempts >= 3) {
    bankSystem.lockAccount(currentCard.getAccountId()); // bank records lock
    atm.returnCard();
    atm.setState(atm.getIdleState());
}
```

**Gap dimension:** Requirements (Dimension 1) — missed during requirements phase

---

## Mistake 5: BankSystem as Part of ATM State (Wrong Layer)

**What happens:** ATM state classes call `account.debit()` directly on an Account object they hold locally. Account is owned by the ATM.

**Why it's wrong:** Account is owned by the bank, not the ATM. The ATM should only communicate with the bank through an API boundary. Multiple ATMs would then have different account objects — divergent balance.

**Fix:** `BankSystem` as a separate service; ATM states call `bankSystem.debit(accountId, amount)`. BankSystem handles account persistence and locking.

**Gap dimension:** Entity Modeling / Architecture (Dimension 2)

---

## Problem-Specific Gotchas

- PIN attempt counter resets when card is ejected (not across sessions)
- ATM's CashDispenser tracks available denominations — $100 bills only gets you even $100 amounts
- The "reserve then execute" pattern is cleaner than "execute then compensate" for interviews
- Depositing cash increments ATM's cash inventory AND credits account — symmetric to withdrawal

---

## Self-Assessment Checklist

- [ ] ATM modeled as state machine (not boolean flags)?
- [ ] Withdrawal checks both balance AND ATM cash before executing either?
- [ ] PIN attempt counter scoped to CardInsertedState (not ATM)?
- [ ] Account locked at bank after 3 failed PINs?
- [ ] Transaction as interface (not if-else in state)?
- [ ] BankSystem as separate entity (not merged into ATM or account)?
- [ ] CashDispenser as separate entity (not just ATM.cashAvailable integer)?

---

## Remediation Targets

| Dimension | If you scored 0–1 | Go to |
|-----------|------------------|-------|
| Design Patterns | Flag soup, no State | GAP_REMEDIATION.md#gap-3 — state trigger: "behavior differs per mode" |
| Entity Modeling | No CashDispenser, no BankSystem boundary | GAP_REMEDIATION.md#gap-2 — noun-verb extraction |
| SOLID | Transactions as if-else | GAP_REMEDIATION.md#gap-4 — OCP violation hunt |
| Concurrency | No atomic withdraw | GAP_REMEDIATION.md#gap-6 — shared state audit |
