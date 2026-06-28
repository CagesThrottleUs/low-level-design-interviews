# Mock Interview Script — Data Exporter

**For the interviewer.**

---

## Opening

> "Design a data export system that exports reports in CSV, JSON, and XML formats. All formats fetch data from the same source and write to a file, but the formatting step differs per format. Ask clarifying questions before designing."

| If candidate asks | Answer |
|------------------|--------|
| Same data source for all formats? | Yes — `fetchData()` is shared |
| Can subclasses reorder steps? | No — fetch → format → write is always the sequence |
| Optional steps per format? | Yes — some formats may need a cleanup step, others don't |
| Runtime format selection? | Yes — caller picks CSV/JSON/XML at runtime |
| Thread safety? | One instance per export request — not shared across threads |
| Streaming large data? | Out of scope — assume data fits in memory |

---

## Phase 2: Core Design — What to Watch For

**Key signal 1: Abstract class, not interface**

Good: "I'll use an abstract class because I need `export()` to be `final` — that enforces step ordering. Interfaces can't have `final` methods."

Bad: `interface DataExporter` with default methods — can't be `final`.

**Key signal 2: `export()` is `final`**

Good: "The template method is `final` — subclasses can only override `formatData()` and `writeOutput()`, not the skeleton."

Bad: `export()` not `final` — subclass can override it and skip steps.

**Key signal 3: Hook methods**

Good: "I'll add a `needsCleanup()` hook method — JSON needs temp file cleanup but CSV doesn't. Making it abstract would force every exporter to implement it. A hook with default `return false` is cleaner."

If candidate makes cleanup `abstract`:
> "What happens when you add a 10th export format that doesn't need cleanup? What does its cleanup method look like?"

---

## Extension Probe (~35 min)

> "Add an Excel export format."

Strong: `ExcelExporter extends DataExporter` — implements `formatData()` and `writeOutput()`. Zero changes to base class or other exporters. If optional steps needed: override hooks.

> "Now the data source is different for PDF — it pulls from an archive, not the live DB."

Strong: `PdfExporter` overrides `fetchData()` (it's `protected`) with archive-specific logic. Base class fetch is still used by all others. Single change, single class.

> "When would you use Strategy instead of Template Method?"

Expected: "Template Method when step ORDER is invariant and you want to enforce it via inheritance. Strategy when the WHOLE algorithm is swappable at runtime via composition. Template Method is compile-time; Strategy is runtime."

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | Fixed step order, optional cleanup hook, shared data source, runtime format selection | |
| Entity Modeling | Abstract base with `final` template, abstract steps, hook methods | |
| Design Patterns | Template Method: `final` skeleton, abstract variable steps, hooks for optional steps | |
| SOLID | OCP: new format = new class; SRP: each exporter does one format | |
| Extensibility | New format = single subclass; no existing code touched | |
| Concurrency | Correctly identifies: no concurrency concern — one instance per request | |
| Communication | Explained WHY abstract class (not interface), WHY `final`, WHY hooks vs abstract | |
| **TOTAL** | | **/21** |
