# Template Method Pattern

**Category:** Behavioral
**LLD interview frequency:** Tier 2
**Problem:** 16 — Data Exporter / Report Generator

---

## The Problem It Solves

When multiple classes implement the same algorithm with minor variations, code is duplicated across them. Changing the shared part means touching all classes.

```java
// Without Template Method — three classes, same structure, duplicated logic
class CsvExporter {
    void export(String period) {
        List<Record> data = fetchFromDatabase(period); // duplicated in all three
        String csv = formatAsCsv(data);
        writeToFile(csv, "report.csv");
        log("CSV export complete");                    // duplicated in all three
    }
}
class JsonExporter {
    void export(String period) {
        List<Record> data = fetchFromDatabase(period); // same fetch code
        String json = formatAsJson(data);
        writeToFile(json, "report.json");
        log("JSON export complete");                   // same logging
    }
}
// If the data source changes, update 3 classes. If logging changes, update 3 classes.
```

---

## Intent

Define the skeleton of an algorithm in a base class, deferring some steps to subclasses. Subclasses can redefine specific steps without changing the algorithm's overall structure.

---

## Trigger Signals

Apply Template Method when you see:
- Multiple classes implementing the same algorithm with only some steps varying
- "The sequence of steps is fixed; only the implementation of steps varies"
- Shared initialisation / teardown logic that must always run
- Problems: data exporters, game frameworks, data migration tools, parsers

---

## Structure

```java
// Abstract class — MUST use abstract class (not interface) for final template method
public abstract class DataExporter {

    // TEMPLATE METHOD — final: subclasses CANNOT reorder or skip steps
    public final void export(String period) {
        List<Record> data = fetchData(period);     // step 1: shared, concrete
        String formatted  = formatData(data);      // step 2: varies — abstract
        writeOutput(formatted);                    // step 3: varies — abstract
        if (needsCleanup()) {                      // hook: optional
            cleanup();
        }
    }

    // Concrete shared step — all subclasses inherit this
    protected List<Record> fetchData(String period) {
        // shared database call
        return repository.findByPeriod(period);
    }

    // Abstract steps — MUST be implemented by each subclass
    protected abstract String formatData(List<Record> data);
    protected abstract void writeOutput(String content);

    // Hooks — subclass MAY override; default = no-op
    protected boolean needsCleanup() { return false; }
    protected void cleanup() { }
}

// Concrete subclass — implements only what varies
public class CsvExporter extends DataExporter {
    @Override
    protected String formatData(List<Record> data) {
        StringBuilder sb = new StringBuilder("id,name,value\n");
        data.forEach(r -> sb.append(r.getId()).append(",")
                             .append(r.getName()).append(",")
                             .append(r.getValue()).append("\n"));
        return sb.toString();
    }

    @Override
    protected void writeOutput(String content) {
        Files.writeString(Path.of("report.csv"), content);
    }
}

// Adding Excel: one new class, zero changes to base or other exporters
public class ExcelExporter extends DataExporter {
    @Override protected String formatData(List<Record> data) { /* xlsx logic */ }
    @Override protected void writeOutput(String content) { /* write xlsx */ }
}
```

---

## Abstract Steps vs Hooks

| Type | Declaration | Purpose |
| --- | --- | --- |
| Abstract step | `protected abstract void step()` | MUST override — step always varies |
| Concrete step | `protected void step() { /* impl */ }` | Shared logic — subclass may override for special cases |
| Hook | `protected boolean needsCleanup() { return false; }` | OPTIONAL override — only for subclasses that need it |

**Why hooks instead of abstract for optional steps:**
Making an optional step `abstract` forces every subclass to implement a no-op method. A hook with a default no-op is cleaner — subclasses that don't need the step ignore it entirely.

---

## Why Abstract Class, Not Interface

`final` methods cannot be declared on interfaces (even Java 8+ `default` methods cannot be `final`). The template method must be `final` to enforce step ordering — only abstract classes support this combination.

```java
interface DataExporter {
    default void export(String period) { ... } // can't be final — subclass can override
}
// If subclass overrides export(), step ordering is broken
```

---

## Template Method vs Strategy

| | Template Method | Strategy |
| --- | --- | --- |
| Mechanism | Inheritance | Composition |
| Step order | Fixed (enforced by `final`) | Flexible — whole algorithm swappable |
| Binding | Compile-time | Runtime |
| Use when | Steps are fixed, some implementations vary | Whole algorithm varies per context |
| Requires | Abstract class | Interface |

When format selection must happen at runtime (user picks CSV/JSON from UI), combine both:

```java
public abstract class DataExporter {
    // Template method enforces step order
    public final void export(String period) {
        List<Record> data = fetchData(period);
        String formatted = formatData(data); // calls abstract method
        writeOutput(formatted);
    }
    protected abstract String formatData(List<Record> data);
    protected abstract void writeOutput(String content);
}
// OR: inject ExportFormatter strategy into a single concrete DataExporter
```

---

## Underlying Principle

**OCP:** Adding a new format = one new subclass. Base class never changes.
**DRY:** `fetchData()` and logging logic live in one place. One class to update when either changes.
**Hollywood Principle:** "Don't call us, we'll call you" — the base class controls the flow; subclasses provide implementations when called.

---

## Real-World Usage

- **`javax.servlet.HttpServlet`** — `service()` is the template; `doGet()`, `doPost()` are abstract steps
- **`java.io.InputStream`** — `read(byte[], int, int)` is the template; `read()` is the abstract step
- **`java.util.AbstractList`** — `get()` and `size()` are abstract; `iterator()`, `contains()` are template methods built on them
- **Spring's `JdbcTemplate`** — `execute()` is the template; query callback is the variable step
- **JUnit's test lifecycle** — `@BeforeEach`, test method, `@AfterEach` form a template

---

## Common Mistakes

- `export()` not declared `final` — subclass overrides and skips steps
- Using `interface` instead of `abstract class` — can't have `final` methods
- Hook methods declared `private` — can't be overridden
- Making all steps `abstract` including optional ones — forces no-op implementations everywhere
- Shared `fetchData()` duplicated in each subclass — defeats the pattern

---

## When NOT to Use

- The whole algorithm varies, not just some steps — use Strategy (composition) instead
- Only two variants exist — just use two classes with a shared utility method
- Step order changes between subclasses — Template Method can't handle this; use Strategy
