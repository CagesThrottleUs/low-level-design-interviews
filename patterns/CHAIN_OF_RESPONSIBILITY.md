# Chain of Responsibility Pattern

**Category:** Behavioral
**LLD interview frequency:** Tier 2
**Problem:** 13 — Logging Framework

---

## The Problem It Solves

When multiple handlers might process a request, but you don't want the sender to know which one handles it — and you want to change the chain at runtime.

```java
// Without CoR — monolithic handler with nested conditions
void handleLogMessage(LogLevel level, String message) {
    if (level == DEBUG) { writeToConsole(message); }
    if (level == WARN || level == ERROR) {
        writeToConsole(message);
        writeToFile(message);
    }
    if (level == ERROR || level == FATAL) {
        writeToConsole(message);
        writeToFile(message);
        writeToDatabase(message);
    }
    // Adding a new level or a new handler means editing this method
}
```

---

## Intent

Avoid coupling the sender of a request to its receivers by giving multiple handlers a chance to handle it. Chain the handlers and pass the request along until one (or all) handle it.

---

## Trigger Signals

Apply Chain of Responsibility when you see:
- "Multiple objects might handle a request, but the sender shouldn't know which"
- "Order of handlers matters; it should be configurable at runtime"
- "Each handler decides whether to process and/or pass on"
- Problems: logging framework, middleware pipeline, support escalation, approval workflows

---

## Two Valid Variants

**Variant 1 — Stop at first handler (strict CoR):**
Each handler either fully handles the request OR passes to the next. First match wins.
Good for: support escalation (L1 → L2 → engineering), approval workflows.

**Variant 2 — Fan-out (all matching handlers fire):**
Each handler checks if it should process, then ALWAYS passes to the next. All qualifying handlers execute.
Good for: logging frameworks (console + file + DB can all receive the same message).

Interviewers accept both — you must explain which you chose and why.

---

## Structure

```java
// Handler interface
public interface LogAppender {
    void append(LogRecord record);
}

// Abstract handler — owns threshold check + next reference
public abstract class AbstractAppender implements LogAppender {
    private volatile LogLevel minLevel;
    private final LogFormatter formatter;

    public final void append(LogRecord record) {
        if (record.getLevel().isAtLeast(minLevel)) {
            write(formatter.format(record)); // process if level matches
        }
        // Fan-out variant: always pass to next (no explicit chain reference needed
        // because Logger iterates the list — simpler than explicit next pointer)
    }

    protected abstract void write(String formatted);
}

// Logger fans out to all appenders (the "chain")
public class Logger {
    private final List<LogAppender> appenders = new CopyOnWriteArrayList<>();

    public void log(LogLevel level, String message) {
        if (!level.isAtLeast(globalMinLevel)) return;
        LogRecord record = new LogRecord(level, message);
        for (LogAppender appender : appenders) {
            try {
                appender.append(record); // each independently decides to handle
            } catch (Exception e) {
                // one failing appender doesn't stop others
            }
        }
    }
}
```

---

## Key Design Decisions in Logging

| Decision | Why |
| --- | --- |
| Level filter per appender, not per Logger | Different destinations need different thresholds |
| Check level BEFORE formatting | Formatting is expensive — skip it for discarded messages |
| Formatter separate from Appender | N×M combinations via composition, not N×M subclasses |
| Independent try-catch per appender | One DB timeout shouldn't stop console logging |
| `CopyOnWriteArrayList` for appenders | Concurrent `addAppender` + `log` is safe |

---

## Underlying Principle

**OCP:** Adding a new appender = new class. Logger never changes.
**SRP:** Each appender handles one destination. Logger handles routing.
**DIP:** Logger depends on `LogAppender` interface, not `ConsoleAppender` directly.

---

## Real-World Usage

- **Java Servlet Filters** — `FilterChain.doFilter()` passes request through filter chain
- **Spring Security** — `FilterSecurityInterceptor` chain
- **Express.js middleware** — `app.use(middleware)` builds a chain
- **Log4j Appenders** — multiple appenders receive the same log event
- **Apache Commons Chain** — explicit CoR implementation

---

## Common Mistakes

- Level filter only on Logger (not per appender) — can't have console=DEBUG and file=WARN+
- Formatting before level check — wasted CPU for discarded messages
- No try-catch per appender — one failure kills all logging
- N×M class explosion from combining format + destination in one class

---

## When NOT to Use

- Only one handler — just call it directly
- Request should be handled by exactly one specific handler — use a lookup map instead
- Handlers need to collaborate (pass results to each other) — use Pipeline/Pipes-and-Filters instead
