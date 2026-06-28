# Mock Interview Script — Logging Framework

**For the interviewer.**

---

## Opening

> "Design a logging framework. Application code should be able to log messages at different severity levels — DEBUG, INFO, WARN, ERROR. These messages should go to multiple destinations like console and file, each with its own minimum level threshold. Ask clarifying questions first."

| If candidate asks | Answer |
|------------------|--------|
| What levels? | DEBUG, INFO, WARN, ERROR, FATAL — ordered by severity |
| Per-destination format? | Yes — console might use plain text, file might use JSON |
| One destination failing stops others? | No — independent |
| Async or sync? | Sync for MVP; async is a secondary extension |
| Named logger hierarchy? | Secondary — single Logger singleton for MVP |
| Runtime level change? | Yes — level should be changeable without restart |

---

## Phase 2: Core Design — What to Watch For

**Key signal 1: Level check before formatting**

Good: "Before I create a LogRecord or call the formatter, I check if the message level is >= the global minimum. Formatting is CPU work — don't do it if the message will be discarded."

**Key signal 2: Level filter on each appender, not only on Logger**

Good: "Each appender has its own minimum level. Logger filters globally (DEBUG+), but console might accept DEBUG+ while file only accepts WARN+."

Bad: "One global threshold on Logger — all appenders get the same filter."

**Key signal 3: Independent appender dispatch**

Good: "I dispatch to each appender inside its own try-catch. One DB appender failing doesn't block the console appender."

**Signal for "CoR vs fan-out":** If candidate implements strict stop-at-first chain:
> "What if I want both the console AND the file to receive a WARN message? Would your design support that?"

---

## Extension Probe (~35 min)

> "Add async logging — the calling thread should return immediately without waiting for file I/O."

Strong: `LinkedBlockingQueue<LogRecord>` buffering records; a single daemon worker thread draining the queue and dispatching to appenders. Mention daemon=true (JVM shouldn't be held alive by logger). Mention shutdown hook to drain before exit.

> "Add a JSON formatter alongside the existing plain-text one."

Strong: `JsonFormatter implements LogFormatter` — zero changes to any appender or Logger. `FileAppender` can be given a `JsonFormatter` while `ConsoleAppender` keeps `SimpleTextFormatter`.

---

## Concurrency Probe (~40 min, if not raised)

> "Thread A and Thread B both call `logger.warn()` at the same time. What could go wrong at the FileAppender?"

Strong: Concurrent writes to `FileWriter` interleave bytes — corrupt log records. Fix: `synchronized` on `FileAppender.write()`. Mentions: `CopyOnWriteArrayList` for appender list (concurrent add/remove); `volatile` on Logger's level field (runtime level change visibility).

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | Per-appender level, independent appender dispatch, runtime level change | |
| Entity Modeling | LogRecord as immutable value object, formatter separate from appender | |
| Design Patterns | Strategy (formatter), fan-out or CoR (appender dispatch) — explained choice | |
| SOLID | OCP: new appender = new class; formatter/appender composed independently | |
| Extensibility | New format or destination = new class, zero Logger changes | |
| Concurrency | synchronized on FileAppender.write(), CopyOnWriteArrayList, volatile level | |
| Communication | Explained WHY level check before formatting, WHY per-appender threshold | |
| **TOTAL** | | **/21** |
