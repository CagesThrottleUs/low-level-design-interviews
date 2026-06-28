# Solution Guide — Splitwise

**Read after your attempt.**

---

## Entity Map

| Class/Interface | Type | Responsibility |
|----------------|------|---------------|
| `User` | Concrete | id, name; no balance logic here (belongs to BalanceSheet) |
| `Group` | Concrete | Members list; expenses list |
| `Expense` | Concrete | Amount, payer, SplitStrategy, list of Split objects |
| `Split` | Concrete | userId + amount — one user's share of an expense |
| `SplitStrategy` | Interface | `calculateSplits(amount, participants): List<Split>` |
| `EqualSplitStrategy` | Concrete | Divide amount equally; distribute remainder to first users |
| `PercentageSplitStrategy` | Concrete | Validate percentages sum to 100; compute each share |
| `ExactSplitStrategy` | Concrete | Validate amounts sum to total; return as-is |
| `BalanceSheet` | Concrete | `Map<String, Map<String, Long>>` — userId → {userId → netOwed}; positive = owed to you |
| `ExpenseService` | Concrete | Add expense; update BalanceSheet; query balances |

---

## Key Design: Split Strategy

```java
interface SplitStrategy {
    List<Split> calculateSplits(long totalAmountCents, List<User> participants, Object config);
}

class EqualSplitStrategy implements SplitStrategy {
    public List<Split> calculateSplits(long total, List<User> participants, Object config) {
        long perPerson = total / participants.size();
        long remainder = total % participants.size();
        List<Split> splits = new ArrayList<>();
        for (int i = 0; i < participants.size(); i++) {
            long amount = perPerson + (i < remainder ? 1 : 0); // distribute remainder to first N users
            splits.add(new Split(participants.get(i).getId(), amount));
        }
        return splits;
    }
}

class PercentageSplitStrategy implements SplitStrategy {
    public List<Split> calculateSplits(long total, List<User> participants, Object config) {
        List<Integer> percentages = (List<Integer>) config;
        if (percentages.stream().mapToInt(Integer::intValue).sum() != 100)
            throw new IllegalArgumentException("Percentages must sum to 100");
        List<Split> splits = new ArrayList<>();
        for (int i = 0; i < participants.size(); i++) {
            long amount = (total * percentages.get(i)) / 100;
            splits.add(new Split(participants.get(i).getId(), amount));
        }
        return splits;
    }
}

class ExactSplitStrategy implements SplitStrategy {
    public List<Split> calculateSplits(long total, List<User> participants, Object config) {
        List<Long> amounts = (List<Long>) config;
        if (amounts.stream().mapToLong(Long::longValue).sum() != total)
            throw new IllegalArgumentException("Exact amounts must sum to total");
        List<Split> splits = new ArrayList<>();
        for (int i = 0; i < participants.size(); i++) {
            splits.add(new Split(participants.get(i).getId(), amounts.get(i)));
        }
        return splits;
    }
}
```

---

## Balance Sheet Update

Core invariant: when A pays for B, B owes A.

```java
class BalanceSheet {
    // balance[a][b] = amount a is owed by b (positive) or a owes b (negative)
    private final Map<String, Map<String, Long>> balance = new ConcurrentHashMap<>();

    public synchronized void updateBalance(String payerId, String participantId, long amount) {
        if (payerId.equals(participantId)) return; // payer's own share, skip

        // participant owes payer: balance[payer][participant] += amount
        balance.computeIfAbsent(payerId, k -> new HashMap<>())
               .merge(participantId, amount, Long::sum);

        // payer owed by participant: balance[participant][payer] -= amount
        balance.computeIfAbsent(participantId, k -> new HashMap<>())
               .merge(payerId, -amount, Long::sum);
    }

    public Map<String, Long> getBalances(String userId) {
        return balance.getOrDefault(userId, Collections.emptyMap());
    }
}
```

---

## Expense Addition Flow

```java
class ExpenseService {
    private final BalanceSheet balanceSheet;

    public void addExpense(Group group, User payer, long totalCents,
                          List<User> participants, SplitStrategy strategy, Object config) {
        List<Split> splits = strategy.calculateSplits(totalCents, participants, config);
        Expense expense = new Expense(totalCents, payer, participants, splits, strategy);
        group.addExpense(expense);

        for (Split split : splits) {
            balanceSheet.updateBalance(payer.getId(), split.getUserId(), split.getAmount());
        }
    }
}
```

---

## Debt Simplification (Bonus)

Algorithm: Simplify N-person debts into minimum transactions.

```
1. Compute net balance for each person (total owed - total owes)
2. Separate into creditors (positive net) and debtors (negative net)
3. Greedily match: largest creditor with largest debtor
4. Create a transaction for min(creditor, debtor)
5. Repeat until all balances are zero
```

```java
List<Transaction> simplify(BalanceSheet sheet, List<User> users) {
    Map<String, Long> net = computeNetBalances(sheet, users);
    PriorityQueue<Map.Entry<String, Long>> creditors = new PriorityQueue<>(
        (a, b) -> Long.compare(b.getValue(), a.getValue()));
    PriorityQueue<Map.Entry<String, Long>> debtors = new PriorityQueue<>(
        (a, b) -> Long.compare(a.getValue(), b.getValue()));

    for (Map.Entry<String, Long> e : net.entrySet()) {
        if (e.getValue() > 0) creditors.add(e);
        else if (e.getValue() < 0) debtors.add(e);
    }

    List<Transaction> result = new ArrayList<>();
    while (!creditors.isEmpty() && !debtors.isEmpty()) {
        Map.Entry<String, Long> c = creditors.poll();
        Map.Entry<String, Long> d = debtors.poll();
        long settled = Math.min(c.getValue(), -d.getValue());
        result.add(new Transaction(d.getKey(), c.getKey(), settled));
        if (c.getValue() > settled) creditors.add(Map.entry(c.getKey(), c.getValue() - settled));
        if (-d.getValue() > settled) debtors.add(Map.entry(d.getKey(), d.getValue() + settled));
    }
    return result;
}
```

---

## Concurrency

**Shared state:** `BalanceSheet` — concurrent expense additions update the same balance map.
**Solution:** `synchronized` on `updateBalance` or per-user lock for finer granularity.

**Rounding note:** Use **integer cents** throughout. `long totalCents = (long)(dollars * 100)`. Never `double` for money.

---

## Extension Points

| "Now add..." | Change |
|-------------|--------|
| New split type (weight-based) | New `WeightedSplitStrategy implements SplitStrategy` |
| Balance change notification | Observer: `BalanceObserver` interface; EmailNotifier, PushNotifier |
| Multi-currency | Add `Currency` to Expense; conversion in SplitStrategy |
| Expense categories | Add to Expense; filtering in ExpenseService |

---

## What Strong Candidates Do Differently

- Use integer cents immediately (without being asked)
- Handle equal split remainder distribution (not just `total / n`)
- Validate that percentages/exact amounts sum correctly before computing
- Separate payer's own share from the balance update
- Discuss debt simplification algorithm even briefly

## What Average Candidates Miss

- Float arithmetic for money (rounding errors compound)
- Equal split remainder not distributed — total doesn't add up
- Payer included in balance update against themselves (owes themselves)
- No validation on PercentageSplit / ExactSplit input
- BalanceSheet as a simple map without concurrent protection
