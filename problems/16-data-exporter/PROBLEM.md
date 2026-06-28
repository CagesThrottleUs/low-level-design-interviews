# Data Exporter / Report Generator

**Difficulty:** Foundation-Intermediate
**Category:** Template Method Pattern
**Time Box:** 45 min (discussion) / 90 min (machine coding)
**Key Patterns:** Template Method (fixed export skeleton), Strategy (alternative for runtime format selection)
**Asked at:** Amazon, Adobe, SAP, Oracle, ThoughtWorks

---

## Problem Statement

Design a data export system that can export reports in multiple formats — CSV, JSON, and XML. All formats follow the same high-level workflow: fetch data from a source, format the data, and write it to an output. The fetch and write steps are the same across formats; only the formatting step varies.

The Template Method pattern enforces that the export always happens in the correct order, while allowing subclasses to provide format-specific implementations.

---

## Actors

| Actor | Description |
|-------|-------------|
| Caller / Application | Triggers an export by calling `exporter.export(period)` |
| DataExporter (abstract) | Owns the skeleton; calls steps in fixed order |
| Concrete Exporters | CSV, JSON, XML — each implements the formatting step |
| Data Source | Repository/DAO that provides the raw data |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: Supports export to CSV, JSON, and XML formats
- FR2: All formats use the same data source (same `fetchData()`)
- FR3: Export sequence is always: fetch → format → write — subclasses cannot reorder
- FR4: Adding a new format (e.g., Excel) requires only adding one new class
- FR5: Some formats may have optional pre/post steps (e.g., JSON needs temp file cleanup)
- FR6: Template method is declared `final` — subclasses cannot override the skeleton

**Secondary (implement if time allows):**
- FR7: Format-specific `writeOutput()` strategies (write to file, stream, S3)
- FR8: Compression hook — optional step before write
- FR9: Export result includes metadata (record count, export time)

---

## Non-Functional Requirements

- **Correctness:** Steps must always execute in order fetch → format → write
- **Extensibility:** New format = one new class; zero changes to existing exporters or base
- **Reuse:** Shared `fetchData()` in base class; subclasses don't duplicate data access

---

## Constraints and Assumptions

- Data is a `List<Record>` returned by `fetchData(String period)`
- Output is written to console or a file (configurable per exporter)
- Exporter instances are not shared across threads (one instance per export request)
- Out of scope: streaming large datasets, incremental export, async export

---

## Good Clarifying Questions to Ask

1. Does each format always write to the same destination, or can destination vary too?
2. Are there any steps that only some formats need (optional hooks)?
3. Should the export skeleton be final — can subclasses override it?
4. Can formats share the same data source, or do some need different sources?
5. Should we support runtime format selection (configuration-based)?
6. Is error handling per-step or do failures abort the whole export?

---

## Template Method vs Strategy — Know the Difference

| Template Method | Strategy |
|----------------|---------|
| Inheritance | Composition |
| Fixed step order (enforced by `final`) | Algorithm fully replaceable |
| Compile-time binding | Runtime binding |
| Abstract class required | Interface sufficient |
| Subclass overrides specific steps | Client provides whole algorithm |
| Use when: steps are fixed, only some vary | Use when: whole algorithm varies |

Rule of thumb: if the ORDER of steps is the invariant, use Template Method. If the whole algorithm can be swapped, use Strategy.

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **DataExporter** — abstract base class; `export(period)` is `final` (template method); `fetchData()` is concrete shared; `formatData()` is abstract; `writeOutput()` is abstract; `needsCleanup()` + `cleanup()` are hooks (optional override)
- **CsvExporter** — implements `formatData()` (comma-delimited) and `writeOutput()` (writes .csv)
- **JsonExporter** — implements `formatData()` (JSON array) and `writeOutput()` (writes .json); overrides `needsCleanup()` to return true
- **XmlExporter** — implements `formatData()` (XML tags) and `writeOutput()` (writes .xml)
- **Record** — simple domain entity with fields (id, name, value)

</details>
