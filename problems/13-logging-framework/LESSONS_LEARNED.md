# Lessons Learned — Logging Framework

---

## Mistake 1: Level Filter Only on Logger (Not Per Appender)

**What happens:** Logger has one global threshold. All appenders receive the same messages.

**Why it's wrong:** Can't have console=DEBUG and file=WARN+ simultaneously. This is a core requirement.

**Fix:** `volatile LogLevel minLevel` on each appender. `AbstractAppender.append()` checks it before calling `write()`.

**Gap dimension:** Requirements (Dimension 1)

---

## Mistake 2: Formatting Before Level Check

**What happens:** `String formatted = formatter.format(record)` called before checking if the message will be discarded.

**Why it's wrong:** JSON serialization and string formatting are CPU-expensive. Formatting a DEBUG message only to discard it wastes cycles. In high-volume systems, this is measurable overhead.

**Fix:**
```java
public final void append(LogRecord record) {
    if (!record.getLevel().isAtLeast(minLevel)) return; // cheap int compare
    write(formatter.format(record));                     // expensive — only if passes
}
```

**Gap dimension:** Requirements / Communication (Dimensions 1, 7)

---

## Mistake 3: Missing `flush()` on FileWriter

**What happens:** `writer.write(text)` called but no `writer.flush()`. Logs appear to be working but records stay in the OS buffer. On JVM exit or crash, buffered records are lost.

**Fix:**
```java
protected synchronized void write(String formatted) {
    writer.write(formatted + System.lineSeparator());
    writer.flush(); // critical — force to disk after every record
}
```

**Gap dimension:** Entity Modeling / Correctness (Dimension 2)

---

## Mistake 4: ArrayList for Appenders — ConcurrentModificationException

**What happens:** `List<LogAppender> appenders = new ArrayList<>()`. `log()` iterates the list. Simultaneously, another thread calls `addAppender()`. `ConcurrentModificationException` thrown.

**Fix:** `CopyOnWriteArrayList` — safe concurrent iteration. Writes copy the array — fine because `addAppender` is rare.
```java
private final List<LogAppender> appenders = new CopyOnWriteArrayList<>();
```

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 5: One Failing Appender Kills All Others

**What happens:** DB appender times out and throws. Exception propagates up and console/file appenders never run.

**Fix:**
```java
for (LogAppender appender : appenders) {
    try {
        appender.append(record);
    } catch (Exception e) {
        System.err.println("Appender failed: " + e.getMessage());
        // continue — other appenders still run
    }
}
```

**Gap dimension:** Requirements / Entity Modeling (Dimensions 1, 2)

---

## Mistake 6: N×M Class Explosion — Hardcoded Formatter in Appender

**What happens:** `ConsoleTextAppender`, `ConsoleJsonAppender`, `FileTextAppender`, `FileJsonAppender` — 4 classes for 2 destinations × 2 formats. With 3 formats × 3 destinations = 9 classes.

**Why it's wrong:** Format and destination vary independently — this calls for composition, not inheritance.

**Fix:** `Formatter` injected into `AbstractAppender` via constructor:
```java
new ConsoleAppender(LogLevel.DEBUG, new JsonFormatter());
new FileAppender(LogLevel.WARN, new SimpleTextFormatter(), "app.log");
// 2 classes total, not 4
```

**Gap dimension:** SOLID / Design Patterns (Dimensions 4, 3)

---

## Self-Assessment Checklist

- [ ] Does each appender have its own level threshold (not just Logger)?
- [ ] Is `formatter.format()` called AFTER the level check?
- [ ] Does `FileAppender.write()` call `flush()` after each record?
- [ ] Is the appender list a `CopyOnWriteArrayList`?
- [ ] Does each appender dispatch have its own try-catch?
- [ ] Is `Formatter` injected into appender (not hardcoded)?
- [ ] Is Logger's level field `volatile` (for runtime changes)?

---

## Remediation Targets

| Dimension | If scored 0–1 | Go to |
|-----------|--------------|-------|
| Requirements | Per-appender level missing | GAP_REMEDIATION.md#gap-1 — requirements drill |
| Design Patterns | N×M class explosion | GAP_REMEDIATION.md#gap-3 — Strategy trigger: "vary independently" |
| SOLID | OCP violation: new appender requires Logger change | GAP_REMEDIATION.md#gap-4 |
| Concurrency | ArrayList, no synchronized on FileWriter | GAP_REMEDIATION.md#gap-6 |
