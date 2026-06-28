# Lessons Learned — In-Memory File System

---

## Mistake 1: List Instead of Map for Directory Children

**What happens:** `List<FileSystemComponent> children` — finding a child by name requires O(n) linear scan.

**Why it's wrong:** Path resolution calls `getChild(name)` for every segment of every path. With deep paths and large directories this becomes quadratic. Interviewers explicitly probe this.

**Fix:** `Map<String, FileSystemComponent>` — O(1) lookup by name:
```java
private final Map<String, FileSystemComponent> children = new LinkedHashMap<>();
// LinkedHashMap preserves insertion order (nice for ls output)
```

**Gap dimension:** Entity Modeling (Dimension 2)

---

## Mistake 2: Storing Full Absolute Path in Each Node

**What happens:** `FileSystemComponent` stores `String absolutePath`. Reconstructed and stored at creation time.

**Why it's wrong:** When `mv` renames a directory, every child's `absolutePath` must be updated recursively. Easy to forget, creating stale paths.

**Fix:** Store only `name` + `parent` reference. Reconstruct full path lazily in `pwd()` by walking up to root:
```java
public String pwd() {
    Deque<String> parts = new ArrayDeque<>();
    FileSystemComponent node = currentDir;
    while (node.getParent() != null) {
        parts.push(node.getName());
        node = node.getParent();
    }
    // join with "/"
}
```

**Gap dimension:** Entity Modeling (Dimension 2)

---

## Mistake 3: `ls` Throws on File Path

**What happens:** `ls("/home/user/file.txt")` throws `NotADirectoryException`.

**Why it's wrong:** Real `ls` on a file path returns just that file's name. This is an explicit requirement in LeetCode #588 and confirmed in multiple interview reports. Missing it loses a requirements point.

**Fix:**
```java
FileSystemComponent target = resolve(path);
if (!target.isDirectory()) return List.of(target.getName()); // file: return itself
return ((Directory) target).ls(); // directory: return sorted children
```

**Gap dimension:** Requirements (Dimension 1)

---

## Mistake 4: `split("/")` Not Handling Empty Strings

**What happens:** `"/home/user".split("/")` → `["", "home", "user"]`. The empty string at index 0 is passed to `getChild("")` → NPE or wrong traversal.

**Why it's wrong:** Silent bug that only surfaces on absolute paths. Relative paths don't trigger it, so candidate may not catch it in testing.

**Fix:**
```java
for (String part : path.split("/")) {
    if (part.isEmpty() || part.equals(".")) continue; // skip empties
    if (part.equals("..")) { /* go up */ continue; }
    // resolve part
}
```

**Gap dimension:** Entity Modeling / Correctness (Dimension 2)

---

## Mistake 5: No Composite Pattern — `instanceof` Everywhere

**What happens:**
```java
int totalSize = 0;
for (FileSystemComponent c : children) {
    if (c instanceof File) totalSize += ((File) c).getContentLength();
    else if (c instanceof Directory) totalSize += computeDirectorySize((Directory) c);
}
```

**Why it's wrong:** Adding a new component type (symlink) requires updating every `instanceof` block. OCP violation.

**Fix:** Put `size()` on `FileSystemComponent`:
```java
abstract class FileSystemComponent {
    public abstract int size(); // leaf and composite both implement
}
// Directory.size() calls child.size() recursively — no instanceof needed
```

**Gap dimension:** Design Patterns + SOLID (Dimensions 3, 4)

---

## Self-Assessment Checklist

- [ ] Children stored in `Map` (not `List`)?
- [ ] Node stores `name` + `parent` ref (not full path string)?
- [ ] `ls` returns `[filename]` for file paths (not error)?
- [ ] `mkdir` is idempotent (no error if exists)?
- [ ] `split("/")` handles empty strings (`if part.isEmpty() continue`)?
- [ ] `size()` is polymorphic — no `instanceof` in the size calculation?
- [ ] `find()` is a DFS on the directory tree?

---

## Remediation Targets

| Dimension | If scored 0–1 | Go to |
|-----------|--------------|-------|
| Design Patterns | `instanceof` chains, no base class | GAP_REMEDIATION.md#gap-3 — Composite: "treat tree nodes uniformly" |
| Entity Modeling | `List` for children, full path stored | GAP_REMEDIATION.md#gap-2 |
| Requirements | `ls` on file throws, `mkdir` not idempotent | GAP_REMEDIATION.md#gap-1 — requirements drill |
| Concurrency | Unsynchronized children map | GAP_REMEDIATION.md#gap-6 |
