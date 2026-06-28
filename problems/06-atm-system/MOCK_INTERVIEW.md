# Mock Interview Script — ATM System

**For the interviewer.**

---

## Opening

> "Design an ATM system. Users insert a card, authenticate with a PIN, and can perform transactions like withdrawing cash and checking their balance. Ask any clarifying questions before designing."

| If candidate asks | Answer |
|------------------|--------|
| Transaction types? | Withdraw and balance inquiry required; deposit is secondary |
| PIN attempts before lockout? | 3 failed attempts locks the card |
| ATM tracks own cash? | Yes — ATM has limited cash inventory |
| Concurrent accounts at two ATMs? | Yes — same account usable at two ATMs simultaneously |
| Atomicity requirement? | Yes — debit and dispense must succeed or fail together |
| Persistence? | Assume a BankSystem that holds accounts; in-memory for ATM state |
| Denominations? | $20, $50, $100 — ATM dispenses in these denominations |

---

## Phase 2: Core Design — What to Watch For

**Key signal 1:** State pattern usage
- Good: "ATM has distinct states — Idle, CardInserted, Authenticated, TransactionInProgress"
- Bad: "I'll use booleans like `isCardInserted`, `isPinVerified`..."

**Key signal 2:** Atomic withdrawal
- Good: "Debit and dispense must be atomic — either both happen or neither"
- Bad: Debit and dispense as two separate methods with no atomicity discussion

**Key signal 3:** Concurrency on account balance
- Good: "Same account at two ATMs → race condition on balance; need locking at BankSystem level"

If stuck on states:
> "What can a customer do when they first approach the ATM? What changes after they insert a card? After they enter the correct PIN?"

If stuck on withdrawal atomicity:
> "What happens if we debit the account successfully but then the ATM runs out of cash — or the machine loses power mid-dispense?"

---

## Extension Probe (~35 min)

> "How would you add a deposit feature?"

Strong: `DepositTransaction implements Transaction` — new class, zero changes to state machine or core. Cash counter increments instead of decrement. Account credit instead of debit.

> "Now add transfer funds to another account."

Strong: `TransferTransaction implements Transaction` — involves two account operations. Mentions two-phase approach: debit source, credit destination. Notes failure handling if credit fails.

---

## Concurrency Probe (~40 min)

> "A customer uses their card at ATM-A while their spouse uses a duplicate card at ATM-B simultaneously. Both try to withdraw $500 from a $700 account. What happens?"

Strong: Both ATMs request authorization from BankSystem simultaneously. Race on balance check + debit. BankSystem must serialize these using synchronized methods or DB-level transactions. Mention optimistic locking or "check-and-set" pattern.

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | PIN lockout, cash inventory, atomicity, concurrent account access | |
| Entity Modeling | ATM states, CashDispenser as entity, BankSystem as boundary, Transaction hierarchy | |
| Design Patterns | State on ATM lifecycle, Strategy/polymorphism on transaction types | |
| SOLID | No god ATM class; transaction logic in Transaction hierarchy | |
| Extensibility | New transaction type = new class | |
| Concurrency | Account balance race at BankSystem level, atomic withdraw | |
| Communication | Explained WHY atomicity needed, WHY State over flags | |
| **TOTAL** | | **/21** |
