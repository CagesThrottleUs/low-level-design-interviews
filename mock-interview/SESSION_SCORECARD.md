# Session Scorecard

Fill out after each practice session. File one per session in `progress/`.

---

```
Date:          ___________
Problem:       ___________
Mode:          Solo / Partner / AI Agent
Time taken:    ___ / 45 min
```

---

## Dimension Scores

### 1. Requirements Gathering ___/3

Notes (what was asked, what was missed):
```

```

### 2. Entity Modeling ___/3

Notes (class structure, anemic vs rich, relationships):
```

```

### 3. Design Patterns ___/3

Notes (patterns applied, justification quality):
```

```

### 4. SOLID Compliance ___/3

Notes (any violations found):
```

```

### 5. Extensibility ___/3

Notes (extension probe — what was asked, how design absorbed it):
```

```

### 6. Concurrency ___/3

Notes (what was identified, solution proposed):
```

```

### 7. Communication ___/3

Notes (narration quality, proactive vs reactive):
```

```

---

## Total: ___/21

---

## Lowest Scoring Dimension

Dimension: ___________
Score: ___/3
Root cause (be specific — e.g., "jumped to code at 2 min", "no interface for pricing"):
```

```

Remediation planned:
```

```

---

## Anti-Patterns Hit

Check all that apply from LESSONS_LEARNED.md for this problem:

- [ ] Anemic domain model (data bags, no behavior)
- [ ] God class / god method
- [ ] No interface extracted (hardcoded implementation)
- [ ] Switch/if-else on type (OCP violation)
- [ ] Float/double for money
- [ ] Missing State pattern (used boolean flags instead)
- [ ] Global lock (not per-resource)
- [ ] Missing race condition identification
- [ ] Silent coder (no narration)
- [ ] No extension probe absorbed cleanly
- [ ] Other: ___________

---

## Comparison to Solution

Key things solution did that you didn't:
```
1.
2.
3.
```

Key things you did correctly:
```
1.
2.
```

---

## Next Session Plan

Problem: ___________
Focus dimension: ___________
Drill to run before next session: GAP_REMEDIATION.md#gap-___
