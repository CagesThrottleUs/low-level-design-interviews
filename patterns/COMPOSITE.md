# Composite Pattern

**Category:** Structural
**LLD interview frequency:** Tier 2
**Problem:** 12 — In-Memory File System

---

## The Problem It Solves

When you have a tree structure and client code needs to treat leaves and composites identically.

```java
// Without Composite — instanceof checks everywhere
int getTotalSize(Object entry) {
    if (entry instanceof File) {
        return ((File) entry).getContentLength();
    } else if (entry instanceof Directory) {
        int total = 0;
        for (Object child : ((Directory) entry).getChildren()) {
            total += getTotalSize(child); // recursive, but ugly
        }
        return total;
    }
    throw new IllegalArgumentException("Unknown type");
}
// Adding SymbolicLink means adding another instanceof branch everywhere
```

---

## Intent

Compose objects into tree structures. Let clients treat individual objects (leaves) and compositions (composites) uniformly through a shared interface.

---

## Trigger Signals

Apply Composite when you see:
- A tree or hierarchy where nodes and leaves should behave the same way
- Client code repeating `instanceof` checks to distinguish leaves from containers
- "Treat a group of objects the same as a single object"
- Problems: file system, org chart, UI component trees, JSON/XML parsers

---

## Structure

```java
// Component — shared interface for leaf and composite
public abstract class FileSystemComponent {
    protected String name;
    protected FileSystemComponent parent;

    public abstract int size();           // uniform operation
    public abstract boolean isDirectory();
    public abstract void print(String indent);
}

// Leaf — no children; does actual work
public class File extends FileSystemComponent {
    private StringBuilder content;

    @Override public int size() { return content.length(); }
    @Override public boolean isDirectory() { return false; }
}

// Composite — holds children; delegates to them
public class Directory extends FileSystemComponent {
    private Map<String, FileSystemComponent> children = new LinkedHashMap<>();

    @Override
    public int size() {
        // Recursive delegation — no instanceof needed
        int total = 0;
        for (FileSystemComponent child : children.values())
            total += child.size();
        return total;
    }

    @Override public boolean isDirectory() { return true; }

    public void addChild(FileSystemComponent c) { children.put(c.getName(), c); }
}

// Client — never checks the type
FileSystemComponent root = fileSystem.resolve("/home");
int size = root.size(); // works on File or Directory without the caller knowing
```

---

## The Composite Insight

Client calls `component.size()` without knowing or caring if it's a leaf or a composite. The composite recursively delegates to its children; each child does the same. The tree traversal is implicit in the object structure — no external traversal logic needed.

---

## Key Implementation Decisions

| Decision | Correct choice | Why |
| --- | --- | --- |
| Children storage | `Map<String, Component>` | O(1) lookup by name during path resolution |
| Parent reference | Store in component | Enables `pwd()` without storing full paths |
| `ls` on file | Return `[filename]` | Real `ls` semantics; failing is a requirements gap |
| `mkdir` idempotency | Silent no-op if exists | Unix semantics; throwing is a bug |

---

## Underlying Principle

**OCP:** Adding a new component type (SymbolicLink) = new class. No existing `instanceof` chains to update.
**Polymorphism:** The entire power of Composite is in `component.method()` dispatching correctly without the caller needing type knowledge.

---

## Real-World Usage

- **Java AWT/Swing:** `Component` base class; `Container` (composite) holds `Component[]`
- **XML/HTML DOM:** `Node` interface; `Element` (composite) holds child nodes
- **JSON parsing:** `JsonValue` base; `JsonObject` (composite) holds key-value pairs
- **JUnit 5:** `TestSuite` (composite) contains `TestCase` leaves

---

## Common Mistakes

- Using `List` instead of `Map` for children — O(n) lookup per path segment
- Storing full absolute path in each node — breaks on rename/move
- Not handling `ls` on file path (must return `[filename]`, not error)
- Forgetting `split("/")` produces empty strings on absolute paths

---

## When NOT to Use

- Hierarchy has only one level (no nesting) — flat list is simpler
- Leaf and composite operations are fundamentally different — forced uniform interface causes confusion
