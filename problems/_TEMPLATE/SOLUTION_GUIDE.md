# Solution Guide — [Problem Name]

**Read after your attempt.** Spoilers ahead.

---

## Entity Map

| Class/Interface | Type | Responsibility |
|----------------|------|---------------|
| [Name] | Interface/Abstract/Concrete | [single responsibility] |
| ... | ... | ... |

---

## Class Diagram (text)

```
[Interface/AbstractClass]
    └── [ConcreteClass A]
    └── [ConcreteClass B]

[ClassX] ──has-a──> [ClassY]
[ClassX] ──has-many──> [ClassZ]
```

---

## Design Patterns Applied

### Pattern 1: [Pattern Name]

**Where:** [which classes]
**Why:** [specific reason this pattern fits]
**Benefit:** [what becomes easier]

```java
// Core interface
interface [Name] {
    [method signature];
}

// Concrete implementation A
class [ConcreteA] implements [Name] {
    public [return type] [method]([params]) {
        // ...
    }
}
```

---

## Core Algorithm / Key Logic

[Explain the most non-obvious logic — state transitions, allocation algorithm, etc.]

```java
// Key algorithm
[code snippet]
```

---

## Concurrency Handling

**Shared mutable state identified:**
- [state 1] — held by [class], concurrent risk: [race condition]
- [state 2] — ...

**Solution:**
```java
// Example: synchronized per spot
public synchronized boolean park(Vehicle v) {
    if (occupied) return false;
    this.parkedVehicle = v;
    this.occupied = true;
    return true;
}
```

---

## Extension Points

| "Now add..." | Change required | Pattern enabling it |
|-------------|----------------|---------------------|
| New [type] | Add new class, zero existing changes | Strategy |
| New [channel] | Add new observer, zero existing changes | Observer |
| New [state] | Add new state class | State |

---

## Full Code Sketch

```java
// [Key interfaces and classes — abbreviated, readable version]
```

---

## What Strong Candidates Do Differently

- [Distinction 1]
- [Distinction 2]
- [Distinction 3]

## What Average Candidates Miss

- [Miss 1]
- [Miss 2]
- [Miss 3]
