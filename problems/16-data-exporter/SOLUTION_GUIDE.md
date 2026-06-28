# Solution Guide — Data Exporter

**Read after your attempt.**

---

## Entity Map

| Class | Type | Responsibility |
|-------|------|---------------|
| `DataExporter` | Abstract class | Owns `export()` as `final` template; `fetchData()` concrete; `formatData()` + `writeOutput()` abstract; `needsCleanup()` hook |
| `CsvExporter` | Concrete | CSV formatting + file write |
| `JsonExporter` | Concrete | JSON formatting + file write; overrides cleanup hook |
| `XmlExporter` | Concrete | XML formatting + file write |
| `Record` | Value object | Domain data being exported |

---

## Core Pattern: Why Abstract Class, Not Interface

`export()` must be `final` to enforce step ordering — interfaces cannot have `final` methods (even Java 8 `default` methods cannot be final). Abstract class is mandatory here.

```java
public abstract class DataExporter {

    // TEMPLATE METHOD — final: subclasses CANNOT reorder or skip steps
    public final void export(String period) {
        List<Record> data = fetchData(period);      // step 1: shared
        String formatted  = formatData(data);       // step 2: abstract — subclass
        writeOutput(formatted);                     // step 3: abstract — subclass
        if (needsCleanup()) {                       // hook: optional
            cleanup();
        }
    }

    // Concrete shared step — all formats use the same data source
    protected List<Record> fetchData(String period) {
        System.out.println("Fetching data for period: " + period);
        return List.of(
            new Record("1", "Alpha", 100.0),
            new Record("2", "Beta",  200.0),
            new Record("3", "Gamma", 300.0)
        );
    }

    // Abstract — MUST be implemented by each format
    protected abstract String formatData(List<Record> data);
    protected abstract void   writeOutput(String content);

    // Hooks — subclass MAY override; default = no cleanup needed
    protected boolean needsCleanup() { return false; }
    protected void    cleanup()      { System.out.println("Default cleanup"); }
}
```

---

## Concrete Exporters

```java
public class CsvExporter extends DataExporter {
    @Override
    protected String formatData(List<Record> data) {
        StringBuilder sb = new StringBuilder("id,name,value\n");
        for (Record r : data) {
            sb.append(r.getId()).append(",")
              .append(r.getName()).append(",")
              .append(r.getValue()).append("\n");
        }
        return sb.toString();
    }

    @Override
    protected void writeOutput(String content) {
        System.out.println("=== CSV Output ===\n" + content);
        // In production: Files.writeString(Path.of("report.csv"), content);
    }
}

public class JsonExporter extends DataExporter {
    @Override
    protected String formatData(List<Record> data) {
        StringBuilder sb = new StringBuilder("[\n");
        for (int i = 0; i < data.size(); i++) {
            Record r = data.get(i);
            sb.append(String.format(
                "  {\"id\":\"%s\",\"name\":\"%s\",\"value\":%s}%s\n",
                r.getId(), r.getName(), r.getValue(),
                i < data.size() - 1 ? "," : ""));
        }
        return sb.append("]").toString();
    }

    @Override
    protected void writeOutput(String content) {
        System.out.println("=== JSON Output ===\n" + content);
    }

    // Override hook — JSON exporter needs temp file cleanup
    @Override protected boolean needsCleanup() { return true; }
    @Override protected void cleanup() {
        System.out.println("JSON-specific: removing temp files");
    }
}

public class XmlExporter extends DataExporter {
    @Override
    protected String formatData(List<Record> data) {
        StringBuilder sb = new StringBuilder("<?xml version=\"1.0\"?>\n<records>\n");
        for (Record r : data) {
            sb.append(String.format(
                "  <record><id>%s</id><name>%s</name><value>%s</value></record>\n",
                r.getId(), r.getName(), r.getValue()));
        }
        return sb.append("</records>").toString();
    }

    @Override
    protected void writeOutput(String content) {
        System.out.println("=== XML Output ===\n" + content);
    }
}

// Domain entity
public final class Record {
    private final String id, name;
    private final double value;
    public Record(String id, String name, double value) {
        this.id = id; this.name = name; this.value = value;
    }
    public String getId()    { return id; }
    public String getName()  { return name; }
    public double getValue() { return value; }
}
```

---

## Client — Runtime Format Selection

```java
public class ExportClient {
    public static void main(String[] args) {
        String format = args.length > 0 ? args[0] : "csv";

        DataExporter exporter = switch (format) {
            case "csv"  -> new CsvExporter();
            case "json" -> new JsonExporter();
            case "xml"  -> new XmlExporter();
            default     -> throw new IllegalArgumentException("Unknown: " + format);
        };

        exporter.export("Q1-2026");
    }
}
```

---

## Hooks Explained

Hooks are concrete methods in the abstract class with a default no-op implementation. Subclasses may override them to add optional behaviour without being forced to.

```
Template method sequence:
    fetchData()      ← concrete, shared
    formatData()     ← abstract, must override
    writeOutput()    ← abstract, must override
    needsCleanup()   ← hook, defaults to false
        ↳ cleanup()  ← hook, only called if needsCleanup() == true
```

Use hooks when only SOME subclasses need an extra step. Making it abstract forces ALL subclasses to implement even if they don't need it.

---

## Template Method vs Strategy — Applied

If format selection must happen at runtime (user picks CSV/JSON from a dropdown), consider combining:

```java
// Strategy interface
interface ExportFormatter {
    String format(List<Record> data);
}

// Single DataExporter uses injected formatter (Strategy) + owns step order (Template Method)
public class DataExporter {
    private final ExportFormatter formatter;

    public DataExporter(ExportFormatter formatter) {
        this.formatter = formatter;
    }

    public final void export(String period) {
        List<Record> data = fetchData(period);
        String formatted  = formatter.format(data); // Strategy
        writeOutput(formatted);
    }
}
// Swap formatter at runtime without subclassing
```

---

## Extension Points

| "Now add..." | Change |
|-------------|--------|
| Excel format | `ExcelExporter extends DataExporter` — zero base changes |
| Compression before write | `preWrite()` hook between `formatData` and `writeOutput` in skeleton |
| Different data source for PDF | Override `fetchData()` in `PdfExporter` (it's `protected`) |
| Async export | Wrap `export()` call in an `ExecutorService.submit()` at the caller level |

---

## What Strong Candidates Do Differently

- `export()` declared `final` — explains this enforces the invariant
- `abstract class` chosen over `interface` — explains `final` methods can't go on interfaces
- Hooks for optional steps (not force all subclasses to implement cleanup)
- `fetchData()` as concrete shared method — eliminates duplication
- Explicitly compare Template Method vs Strategy for this problem

## What Average Candidates Miss

- `export()` not `final` — subclass can reorder or skip steps
- Using `interface` instead of `abstract class` — then template method can't be final
- Hook methods `private` — can't be overridden
- No shared `fetchData()` — each subclass duplicates data access logic
- Making cleanup abstract — forces every exporter to implement it even if unnecessary
