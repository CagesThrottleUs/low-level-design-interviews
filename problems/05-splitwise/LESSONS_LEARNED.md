# Lessons Learned — Splitwise

---

## Mistake 1: Using Float/Double for Money

**What happens:** Candidate uses `double amount` for expense amounts and shares.

**Why it's wrong:** `10.0 / 3 = 3.3333...` — floating point can't represent all decimal fractions exactly. $10 split 3 ways: each person owes $3.33, but 3 × 3.33 = $9.99 ≠ $10. The missing cent causes running balance errors that compound.

**Fix:** Use integer cents throughout:
```java
// Instead of double amount = 10.0;
long amountCents = 1000; // $10.00 = 1000 cents

// Equal split with remainder distribution:
long perPerson = amountCents / participants.size(); // 333 cents
long remainder = amountCents % participants.size(); // 1 cent
// Give the remainder cent to the first person
```

**Gap dimension:** Entity Modeling / Correctness (Dimension 2)

---

## Mistake 2: Payer Updates Own Balance

**What happens:** When updating BalanceSheet, candidate iterates all participants including the payer and creates a debt entry for payer → payer.

**Why it's wrong:** Payer doesn't owe themselves. Including them creates a false debt entry that makes totals wrong.

**Fix:**
```java
for (Split split : splits) {
    if (split.getUserId().equals(payer.getId())) continue; // skip payer's own share
    balanceSheet.updateBalance(payer.getId(), split.getUserId(), split.getAmount());
}
```

**Gap dimension:** Entity Modeling / Correctness (Dimension 2)

---

## Mistake 3: No SplitStrategy Interface (OCP Violation)

**What happens:** `ExpenseService.addExpense()` has a switch on split type:
```java
if (splitType == EQUAL) { ... }
else if (splitType == PERCENTAGE) { ... }
else if (splitType == EXACT) { ... }
```

**Why it's wrong:** Adding a new split type (by weight, by share count) requires modifying `ExpenseService`. OCP violation.

**Fix:**
```java
interface SplitStrategy {
    List<Split> calculateSplits(long totalCents, List<User> participants, Object config);
}
// Expense holds the strategy:
class Expense {
    private final SplitStrategy splitStrategy;
}
// New type = new class, zero changes to ExpenseService
```

**Gap dimension:** Design Patterns + SOLID (Dimensions 3, 4)

---

## Mistake 4: Equal Split Remainder Not Distributed

**What happens:** `$10 / 3 people = $3.33` each. Candidate stores 3.33 or 333 cents for each. Total = 999 cents ≠ 1000 cents.

**Why it's wrong:** The split amounts don't add up to the expense total. Running balance errors appear in BalanceSheet.

**Fix:** Distribute remainder cents to first N participants:
```java
long remainder = totalCents % participants.size();
for (int i = 0; i < participants.size(); i++) {
    long share = perPerson + (i < remainder ? 1 : 0);
    splits.add(new Split(participants.get(i).getId(), share));
}
```

**Gap dimension:** Correctness / Entity Modeling (Dimension 2)

---

## Problem-Specific Gotchas

- Validation: PercentageSplit must check sum = 100%, ExactSplit must check sum = total — fail fast
- BalanceSheet direction: `balance[A][B] = X` means B owes A `X` cents — get the sign right
- Debt simplification greedy algorithm: O(n log n) with priority queues; mention even if not implemented
- User can be in multiple groups — global BalanceSheet vs per-group BalanceSheet: interviewer may ask
- Concurrency: two expenses added simultaneously to same group update same BalanceSheet entries

---

## Self-Assessment Checklist

- [ ] Are amounts in integer cents (not float)?
- [ ] Is payer excluded from their own debt entry?
- [ ] Is SplitStrategy an interface with 3 implementations?
- [ ] Does equal split distribute the remainder correctly?
- [ ] Does PercentageSplit validate sum = 100%?
- [ ] Does ExactSplit validate sum = total?
- [ ] Is BalanceSheet update atomic/synchronized?
- [ ] Can I add a new split type without editing ExpenseService?

---

## Remediation Targets

| Dimension | If you scored 0–1 | Go to |
|-----------|------------------|-------|
| Entity Modeling | Float money, payer-self-debt, wrong remainder | GAP_REMEDIATION.md#gap-2 — think about object invariants |
| Design Patterns | Switch on split type | GAP_REMEDIATION.md#gap-3 — trigger: "multiple types behave differently" |
| SOLID | OCP violation on split type | GAP_REMEDIATION.md#gap-4 — violation hunt |
| Concurrency | Unsynchronized BalanceSheet | GAP_REMEDIATION.md#gap-6 — shared state audit |
