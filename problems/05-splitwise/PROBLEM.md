# Splitwise (Expense Splitting)

**Difficulty:** Intermediate
**Category:** Business Logic + Group Management
**Time Box:** 45 min (discussion) / 90 min (machine coding)
**Key Patterns:** Strategy (split types), Observer (balance change notifications)
**Asked at:** Amazon, Uber, Razorpay, Flipkart, Gojek, Swiggy

---

## Problem Statement

Design an expense splitting application similar to Splitwise. Users can create groups, add expenses, and the system tracks who owes what to whom. Expenses can be split equally, by percentage, or by exact amounts. The system should minimize the number of transactions required to settle all debts (optional: debt simplification).

---

## Actors

| Actor | Description |
|-------|-------------|
| User | Joins groups; adds expenses; views balances; settles debts |
| Group | Container for members and shared expenses |
| Expense | A transaction paid by one member, shared among others |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: Users can register and join groups
- FR2: Any group member can add an expense (paid by one person, shared by multiple)
- FR3: Support three split types: Equal, Percentage, Exact
- FR4: Track net balance per user (what they owe total, to whom)
- FR5: Show balance summary: who owes whom how much

**Secondary (implement if time allows):**
- FR6: Settle a debt between two users
- FR7: Debt simplification — minimize transaction count to settle all group debts
- FR8: Expense notes and categories
- FR9: Notification when balance changes

---

## Non-Functional Requirements

- **Correctness:** Split amounts must sum to exact expense total (no rounding errors)
- **Concurrency:** Multiple users can add expenses simultaneously
- **Persistence:** In-memory is fine
- **Scale:** Groups up to 100 users; not distributed

---

## Constraints and Assumptions

- One payer per expense (the person who paid the bill)
- Participants can be a subset of the group
- Amounts in integer cents (not float) to avoid rounding issues
- For Percentage split: percentages must sum to 100%
- For Exact split: amounts must sum to total expense
- Out of scope: multi-currency, payment processing, chat

---

## Good Clarifying Questions to Ask

1. What split types do we need? Equal, exact, percentage?
2. Can a non-group-member pay for a group expense?
3. Are amounts in float or integer (cents)?
4. Do we need debt simplification (minimize transactions)?
5. How do we handle rounding in equal splits? (e.g., $10 split 3 ways)
6. Can one person be in multiple groups?
7. Notifications when balance changes?

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **User** — id, name, email; has a global balance map (who owes them what)
- **Group** — id, name, list of members, list of expenses
- **Expense** — amount, payer (User), participants (subset of group), SplitStrategy
- **SplitStrategy** — interface; EqualSplit, PercentageSplit, ExactSplit implement it
- **Split** — a single user's share of an expense (user + amount)
- **BalanceSheet** — tracks net owed between each pair of users; `Map<UserId, Map<UserId, Long>>`
- **ExpenseService** — orchestrates adding expenses, updating balances

</details>
