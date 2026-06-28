# Lessons Learned — Data Exporter

---

## Mistake 1: Using Interface Instead of Abstract Class

**What happens:** `interface DataExporter { default void export() {...} }` — can't mark `export()` as `final`.

**Why it's wrong:** `default` methods on interfaces cannot be `final`. Subclasses can override `export()` and reorder or skip steps. The template method invariant is broken.

**Fix:** Abstract class. `export()` declared `final`. Java's `interface` doesn't allow `final` on default methods.

**Gap dimension:** Design Patterns (Dimension 3)

---

## Mistake 2: `export()` Not Declared `final`

**What happens:** `public void export(String period) { ... }` — subclass overrides the skeleton.

**Why it's wrong:** A subclass can now skip `fetchData()`, call `writeOutput()` before `formatData()`, or add random steps. The entire purpose of Template Method — enforcing step order — is defeated.

**Fix:** `public final void export(String period) { ... }` — `final` makes this non-overridable.

**Gap dimension:** Design Patterns (Dimension 3)

---

## Mistake 3: Making All Steps Abstract (Including Optional Ones)

**What happens:** `protected abstract void cleanup();` — every exporter must implement it, even if it does nothing.

**Why it's wrong:** A `CsvExporter.cleanup()` that does nothing is noise. Worse, a developer might accidentally `return` in it and cause a side effect. Adding a 10th exporter means implementing a no-op cleanup every time.

**Fix:** Hook pattern — concrete method with a default no-op:
```java
protected boolean needsCleanup() { return false; } // hook: default = skip
protected void cleanup() { }                        // hook: no-op default
// Only override in exporters that need cleanup
```

**Gap dimension:** Design Patterns (Dimension 3)

---

## Mistake 4: Each Subclass Duplicates `fetchData()`

**What happens:** Every exporter reimplements the data-fetching logic — same SQL query, same repository call.

**Why it's wrong:** DRY violation. When the query changes, 3 exporters must be updated. One is always missed.

**Fix:** `fetchData()` as a concrete method in the base class. Subclasses inherit it. Only override in the rare case where a format genuinely needs a different source.

**Gap dimension:** SOLID — DRY / SRP (Dimension 4)

---

## Mistake 5: Hook Methods `private` Instead of `protected`

**What happens:** `private boolean needsCleanup() { return false; }` — subclass cannot override it. It's effectively hard-coded.

**Fix:** Hook methods must be `protected`:
```java
protected boolean needsCleanup() { return false; } // protected: subclass can override
```

**Gap dimension:** Entity Modeling (Dimension 2)

---

## Self-Assessment Checklist

- [ ] Is the base class `abstract class` (not `interface`)?
- [ ] Is `export()` declared `final`?
- [ ] Is `fetchData()` a concrete shared method (not abstract)?
- [ ] Are variable steps (`formatData`, `writeOutput`) declared `abstract`?
- [ ] Are optional steps hooks (`protected`, concrete, default no-op) not abstract?
- [ ] Are hooks `protected` (not `private`)?
- [ ] Can a new format be added by adding ONE class with ZERO existing changes?

---

## Template Method vs Strategy Quick Reference

| Aspect | Template Method | Strategy |
|--------|----------------|---------|
| Mechanism | Inheritance | Composition |
| Step order | Fixed (enforced by `final`) | Flexible |
| Binding | Compile-time | Runtime |
| Use when | Steps are fixed, some vary | Whole algorithm swappable |
| Requires | Abstract class | Interface |
| JDK example | `HttpServlet.service()` | `Comparator` |

---

## Remediation Targets

| Dimension | If scored 0–1 | Go to |
|-----------|--------------|-------|
| Design Patterns | Interface used, export() not final, wrong hooks | GAP_REMEDIATION.md#gap-3 — Template Method trigger: "fixed step order, variable steps" |
| SOLID | fetchData() duplicated in each subclass | GAP_REMEDIATION.md#gap-4 — SRP: who owns data access? |
| Entity Modeling | Private hooks, no abstract/concrete distinction | GAP_REMEDIATION.md#gap-2 |
