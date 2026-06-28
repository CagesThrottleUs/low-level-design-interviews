# Mock Interview Script — Splitwise

**For the interviewer.**

---

## Opening

> "Design an expense splitting application — something like Splitwise. Users join groups, add expenses, and the system tracks who owes what. Before designing, ask any clarifying questions."

| If candidate asks | Answer |
|------------------|--------|
| Split types? | Equal, percentage, and exact amounts — all three |
| Integer or float for money? | Good question — use integer cents to avoid rounding issues |
| Multiple groups? | Yes — users can be in multiple groups |
| Debt simplification? | Nice to have; not required for MVP |
| Who can add expenses? | Any group member |
| Persistence? | In-memory is fine |
| Concurrency? | Multiple users can add expenses to the same group simultaneously |

---

## Phase 2: Core Design

**Key signals:**

1. Does candidate use integer cents (not float) for money?
2. Does candidate identify SplitStrategy as an interface?
3. Does candidate handle the payer-excluded-from-their-own-debt edge case?
4. Does candidate handle equal split remainder (11 cents / 3 people)?

If candidate uses float for money:
> "What happens when you split $10 equally among 3 people using floating point?"
> (Expected: 3.333... — ask them to think about how to avoid this)

If no SplitStrategy interface:
> "If we add a new split type — say by weight — how would that change your design?"

---

## Extension Probe (~35 min)

> "How would you add notifications — when someone adds an expense, all other group members get an email?"

Strong: Observer pattern — `BalanceChangeObserver` interface; `EmailNotifier` implements it. `ExpenseService` notifies observers on each expense add. New channel = new observer.

---

## Concurrency Probe (~40 min)

> "Two group members add expenses simultaneously. What's the risk?"

Strong: Race on BalanceSheet update. Concurrent writes to same user pair balance can produce incorrect net amounts. Synchronized BalanceSheet or per-user-pair locking.

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | Split types, integer amounts, payer-vs-participant distinction | |
| Entity Modeling | Expense with SplitStrategy, Split as value object, BalanceSheet as separate class | |
| Design Patterns | Strategy for split types, Observer for notifications | |
| SOLID | OCP: new split type = new class | |
| Extensibility | New split type or notification channel = new class, no existing changes | |
| Concurrency | BalanceSheet update synchronized | |
| Communication | Explained WHY integer cents, WHY SplitStrategy interface | |
| **TOTAL** | | **/21** |
