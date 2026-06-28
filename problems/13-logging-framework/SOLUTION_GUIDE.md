# Solution Guide — Logging Framework

**Read after your attempt.**

---

## Entity Map

| Class/Interface | Type | Responsibility |
|----------------|------|---------------|
| `LogLevel` | Enum | DEBUG(1)…FATAL(5); `isAtLeast(other)` comparison |
| `LogRecord` | Immutable value object | timestamp, level, message, threadName — captured at call site |
| `LogFormatter` | Interface (Strategy) | `format(LogRecord): String` |
| `SimpleTextFormatter` | Concrete | `[timestamp] [thread] [LEVEL] message` |
| `JsonFormatter` | Concrete | `{"timestamp":…, "level":…, "message":…}` |
| `LogAppender` | Interface | `append(LogRecord)` |
| `AbstractAppender` | Abstract | Owns minLevel + formatter; `append()` checks threshold then delegates to `write()` |
| `ConsoleAppender` | Concrete | `write()` → `System.out.println()` |
| `FileAppender` | Concrete | `write()` → `FileWriter.write()` + `flush()` — synchronized |
| `Logger` | Singleton | Global minLevel; `CopyOnWriteArrayList<LogAppender>`; fans out to all appenders |

---

## Core Implementation

```java
public enum LogLevel {
    DEBUG(1), INFO(2), WARN(3), ERROR(4), FATAL(5);
    private final int severity;
    LogLevel(int s) { this.severity = s; }
    public boolean isAtLeast(LogLevel other) { return severity >= other.severity; }
}

public final class LogRecord {
    private final LogLevel level;
    private final String message;
    private final long timestamp = System.currentTimeMillis();
    private final String threadName = Thread.currentThread().getName();

    public LogRecord(LogLevel level, String message) {
        this.level = level; this.message = message;
    }
    public LogLevel getLevel()    { return level; }
    public String getMessage()    { return message; }
    public long getTimestamp()    { return timestamp; }
    public String getThreadName() { return threadName; }
}

// Strategy: formatter
public interface LogFormatter {
    String format(LogRecord record);
}

public class SimpleTextFormatter implements LogFormatter {
    public String format(LogRecord r) {
        return String.format("[%tF %tT] [%s] [%s] %s",
            r.getTimestamp(), r.getTimestamp(), r.getThreadName(),
            r.getLevel(), r.getMessage());
    }
}

public class JsonFormatter implements LogFormatter {
    public String format(LogRecord r) {
        return String.format("{\"ts\":%d,\"level\":\"%s\",\"thread\":\"%s\",\"msg\":\"%s\"}",
            r.getTimestamp(), r.getLevel(), r.getThreadName(), r.getMessage());
    }
}

// Template Method + Strategy: each appender owns its level filter + formatter
public abstract class AbstractAppender implements LogAppender {
    private volatile LogLevel minLevel;     // volatile: runtime change visibility
    private final LogFormatter formatter;

    public AbstractAppender(LogLevel minLevel, LogFormatter formatter) {
        this.minLevel = minLevel; this.formatter = formatter;
    }

    @Override
    public final void append(LogRecord record) {
        if (!record.getLevel().isAtLeast(minLevel)) return; // cheap check first
        write(formatter.format(record)); // expensive only if passes threshold
    }

    protected abstract void write(String formatted);
    public void setMinLevel(LogLevel level) { this.minLevel = level; }
}

public class ConsoleAppender extends AbstractAppender {
    public ConsoleAppender(LogLevel minLevel, LogFormatter formatter) {
        super(minLevel, formatter);
    }
    @Override
    protected synchronized void write(String formatted) {
        System.out.println(formatted);
    }
}

public class FileAppender extends AbstractAppender implements AutoCloseable {
    private final FileWriter writer;

    public FileAppender(LogLevel minLevel, LogFormatter formatter, String path)
            throws IOException {
        super(minLevel, formatter);
        this.writer = new FileWriter(path, true); // append mode
    }

    @Override
    protected synchronized void write(String formatted) {
        try {
            writer.write(formatted + System.lineSeparator());
            writer.flush(); // flush after every record — critical for visibility
        } catch (IOException e) {
            System.err.println("FileAppender error: " + e.getMessage());
        }
    }

    @Override public void close() throws IOException { writer.close(); }
}

// Logger — Singleton; fan-out to all matching appenders independently
public class Logger {
    private static volatile Logger instance;
    private volatile LogLevel globalMinLevel;
    // CopyOnWriteArrayList: safe concurrent iteration + rare add/remove
    private final List<LogAppender> appenders = new CopyOnWriteArrayList<>();

    private Logger(LogLevel level) { this.globalMinLevel = level; }

    public static Logger getInstance(LogLevel level) {
        if (instance == null) synchronized (Logger.class) {
            if (instance == null) instance = new Logger(level);
        }
        return instance;
    }

    public void addAppender(LogAppender a) { appenders.add(a); }
    public void setLevel(LogLevel level)   { this.globalMinLevel = level; } // volatile write

    public void log(LogLevel level, String message) {
        if (!level.isAtLeast(globalMinLevel)) return; // global guard — before anything
        LogRecord record = new LogRecord(level, message);
        for (LogAppender appender : appenders) {
            try {
                appender.append(record); // each appender checks its own threshold
            } catch (Exception e) {
                System.err.println("Appender failed: " + e.getMessage());
                // don't let one broken appender stop others
            }
        }
    }

    public void debug(String msg) { log(LogLevel.DEBUG, msg); }
    public void info(String msg)  { log(LogLevel.INFO, msg); }
    public void warn(String msg)  { log(LogLevel.WARN, msg); }
    public void error(String msg) { log(LogLevel.ERROR, msg); }
    public void fatal(String msg) { log(LogLevel.FATAL, msg); }
}
```

---

## Async Logging Extension

```java
class AsyncLogProcessor implements AutoCloseable {
    private final BlockingQueue<LogRecord> queue = new LinkedBlockingQueue<>(10_000);
    private final ExecutorService worker;
    private final List<LogAppender> appenders;

    AsyncLogProcessor(List<LogAppender> appenders) {
        this.appenders = appenders;
        this.worker = Executors.newSingleThreadExecutor(r -> {
            Thread t = new Thread(r, "async-log-worker");
            t.setDaemon(true); // JVM can exit without waiting for this thread
            return t;
        });
        worker.submit(this::drain);
    }

    public void submit(LogRecord record) {
        // offer() — non-blocking; drops record if queue full (backpressure)
        if (!queue.offer(record)) System.err.println("Log queue full — record dropped");
    }

    private void drain() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                LogRecord record = queue.poll(100, TimeUnit.MILLISECONDS);
                if (record != null) appenders.forEach(a -> a.append(record));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
        // Drain remaining on shutdown
        queue.forEach(r -> appenders.forEach(a -> a.append(r)));
    }

    @Override public void close() {
        worker.shutdown();
        try { worker.awaitTermination(2, TimeUnit.SECONDS); }
        catch (InterruptedException e) { worker.shutdownNow(); }
    }
}
```

---

## Extension Points

| "Now add..." | Change |
|-------------|--------|
| JSON formatter | `JsonFormatter implements LogFormatter` — zero appender changes |
| Kafka appender | `KafkaAppender extends AbstractAppender` — zero Logger changes |
| Async delivery | `AsyncLogProcessor` wraps existing appenders |
| Log rotation | `RollingFileAppender extends AbstractAppender` — checks size/date in `write()` |
| Named loggers | `LogManager` registry + parent chain for level inheritance |

---

## What Strong Candidates Do Differently

- Level check BEFORE creating `LogRecord` — no allocation for discarded messages
- `volatile` on `globalMinLevel` — runtime change visible across threads without synchronization
- `synchronized` on `FileAppender.write()` — per-record atomicity
- `CopyOnWriteArrayList` for appenders — concurrent `addAppender` won't cause `ConcurrentModificationException` during `log()`
- Independent try-catch per appender — one failing appender doesn't stop others
- `formatter.format()` called inside `append()` (inside the level check) — not before

## What Average Candidates Miss

- One global threshold on Logger (not per appender)
- Missing `flush()` on FileWriter — records buffered, appear to be lost
- `ArrayList` for appenders — `ConcurrentModificationException` when appender added during log()
- Formatter hardcoded inside each appender class (N×M explosion when adding JSON to all)
- No exception isolation — one DB appender timeout kills all logging
