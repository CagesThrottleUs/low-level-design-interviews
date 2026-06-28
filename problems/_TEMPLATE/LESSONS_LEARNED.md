# Lessons Learned — [Problem Name]

Common mistakes candidates make on this specific problem. Check yourself after each attempt.

---

## Mistake 1: [Name]

**What happens:** [description]
**Why it's wrong:** [explanation]
**Fix:**
```java
// Before
[bad code]

// After
[good code]
```
**Gap dimension:** [which rubric dimension this maps to]

---

## Mistake 2: [Name]

**What happens:** ...
**Why it's wrong:** ...
**Fix:** ...
**Gap dimension:** ...

---

## Problem-Specific Gotchas

- [Gotcha 1 — unique to this problem]
- [Gotcha 2]
- [Gotcha 3]

---

## Self-Assessment Checklist

After your attempt, verify:

- [ ] Did I identify all actors correctly?
- [ ] Are my classes doing only one thing?
- [ ] Can I add [specific extension] without modifying existing classes?
- [ ] Did I address the concurrent [operation] scenario?
- [ ] Are states/types represented as enums, not magic strings?
- [ ] Is there any god class (Manager, Handler, Controller)?
- [ ] Can I explain WHY I chose each design decision?

---

## Remediation Targets

| Dimension | If you scored 0–1 | Go to |
|-----------|------------------|-------|
| Requirements | Missed [actors/use cases] | GAP_REMEDIATION.md#gap-1 + redo requirements drill on this problem |
| Entity Modeling | Anemic classes, god class | GAP_REMEDIATION.md#gap-2 + redo noun-verb extraction |
| Patterns | No Strategy/Observer/State | GAP_REMEDIATION.md#gap-3 + pattern trigger drill |
| SOLID | OCP violation on [specific thing] | GAP_REMEDIATION.md#gap-4 + violation hunt |
| Extensibility | Rewrite needed for [extension] | GAP_REMEDIATION.md#gap-5 + extension challenge |
| Concurrency | Missed [race condition] | GAP_REMEDIATION.md#gap-6 + shared state audit |
| Communication | Silent coder | GAP_REMEDIATION.md#gap-7 + think-aloud drill |
