# Logging Framework

**Difficulty:** Intermediate
**Category:** Chain of Responsibility + Strategy
**Time Box:** 45 min (discussion) / 90 min (machine coding)
**Key Patterns:** Chain of Responsibility (level-based routing), Strategy (formatter), Observer (fan-out to sinks)
**Asked at:** Google, Amazon, Adobe, Uber, Goldman Sachs

---

## Problem Statement

Design a logging framework similar to Log4j or SLF4J. The system accepts log messages at different severity levels (DEBUG, INFO, WARN, ERROR, FATAL), routes them to appropriate handlers based on level thresholds, and writes them to multiple output destinations (console, file, database) simultaneously. Each destination can have its own minimum log level and output format.

---

## Actors

| Actor | Description |
|-------|-------------|
| Application Code | Calls `logger.info()`, `logger.error()` etc. |
| Logger | Entry point; filters by global threshold; fans out to appenders |
| Appenders | Each listens on a minimum level; formats and writes to its destination |
| Formatter | Converts a log record to a string (plain text, JSON, etc.) |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: Support log levels: DEBUG, INFO, WARN, ERROR, FATAL (ordered by severity)
- FR2: Logger has a configurable minimum level — messages below it are silently discarded
- FR3: Write to multiple destinations simultaneously (console, file, database)
- FR4: Each destination has its own minimum level threshold (e.g., file only writes WARN+)
- FR5: Each destination can have its own formatter (plain text vs JSON)
- FR6: Thread-safe — concurrent log calls from multiple threads must not interleave at the output
- FR7: Extensible — new destinations and formats without modifying existing code

**Secondary (implement if time allows):**
- FR8: Async logging — calling thread is never blocked by I/O
- FR9: Named loggers with level inheritance (Log4j-style dotted hierarchy)
- FR10: Log rotation — file appender rolls over on size or time

---

## Non-Functional Requirements

- **Thread safety:** Per-record atomicity required (no byte interleaving between concurrent log calls)
- **Latency:** Global level check before any formatting — don't format if message will be discarded
- **Extensibility:** New appender = new class; no changes to Logger

---

## Constraints and Assumptions

- Appenders fire independently — one appender throwing should not stop others
- Level check happens before formatting (cheap guard before expensive operation)
- `Logger` is a Singleton for MVP; named loggers are secondary
- Out of scope: log aggregation, distributed tracing, remote log shipping

---

## Good Clarifying Questions to Ask

1. What log levels are needed? Is TRACE below DEBUG needed?
2. Does each destination get its own format, or one global format?
3. Should one destination failing (e.g., DB down) stop others from receiving the message?
4. Sync or async delivery to destinations?
5. Is there a named logger hierarchy (Log4j `com.app.service` inherits from `com.app`)?
6. Should level threshold be changeable at runtime without restart?

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **LogLevel** — enum with numeric severity: DEBUG(1), INFO(2), WARN(3), ERROR(4), FATAL(5)
- **LogRecord** — immutable value object: timestamp, level, message, threadName (captured at construction)
- **LogFormatter** (interface) — `format(LogRecord) → String`; SimpleTextFormatter, JsonFormatter implement it
- **LogAppender** (interface) — `append(LogRecord)`; each has its own minLevel + formatter
- **AbstractAppender** — base class; owns level threshold check + formatter; subclasses implement `write(String)`
- **ConsoleAppender**, **FileAppender**, **DatabaseAppender** — concrete appenders
- **Logger** — Singleton; global minLevel; `List<LogAppender> appenders` (CopyOnWriteArrayList); fans out to all appenders independently

</details>

---

## Two Valid Interpretations of "Chain"

This problem is called Chain of Responsibility but real implementations often use fan-out (all matching appenders fire), not strict single-handler chain. Know both:

**Strict CoR (stops at first match):** Each handler handles OR passes to next. Used when one handler "owns" a request.

**Fan-out (all matching fire):** Logger iterates all appenders; each checks its own threshold; all qualifying ones write. This matches real logging frameworks. Interviewers accept either — but you must explain your choice.
