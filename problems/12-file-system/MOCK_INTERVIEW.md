# Mock Interview Script — In-Memory File System

**For the interviewer.**

---

## Opening

> "Design an in-memory file system. It should support creating files and directories, reading/writing file content, and navigating paths. Ask any clarifying questions before designing."

| If candidate asks | Answer |
|------------------|--------|
| Absolute and relative paths? | Both — absolute starts with `/`, relative from current directory |
| `ls` on file path? | Return `[filename]` — don't error |
| `mkdir` idempotent? | Yes — no error if directory already exists |
| Thread safety? | Yes — multiple processes can read/write concurrently |
| Persistence? | In-memory only |
| Symlinks? | Out of scope for now |
| `rm` recursive? | Yes — `rm` on a directory deletes it and all contents |

---

## Phase 2: Core Design — What to Watch For

**Key signal 1: Composite pattern recognition**

Good: "Files and directories both have a `size()` method — I'll create a `FileSystemComponent` abstract class or interface. File is a leaf, Directory is a composite that delegates `size()` to its children recursively."

Bad: Two completely separate classes with no shared interface. `ls` has to check `instanceof File` or `instanceof Directory` everywhere.

**Key signal 2: `Map` vs `List` for directory children**

Good: `Map<String, FileSystemComponent>` — O(1) lookup by name when resolving paths.

Bad: `List<FileSystemComponent>` — O(n) scan per path segment. With deep paths and large directories, this compounds.

**Key signal 3: `parent` reference in each node**

Good: Each node stores a `parent` reference. `pwd()` reconstructs the path by walking up to root. No need to store the full path string in each node.

Bad: Storing the full absolute path in each node — breaks silently on `mv()` (would need to update paths in all children).

If candidate uses `instanceof` checks everywhere:
> "Is there a way to make the client code not need to know whether it's dealing with a file or a directory?"

---

## Extension Probe (~35 min)

> "Add file permissions. Users with read-only access should not be able to write files."

Strong: `permissions: int` bitmask (9-bit Unix-style rwxrwxrwx) on `FileSystemComponent`. `FileSystem` checks permissions before any write operation. Protection Proxy pattern: `PermissionCheckingFileSystem` wraps `FileSystem`.

> "Add `mv(src, dst)` — move a file or directory."

Strong: Remove from source parent, re-parent to destination directory. Nodes store `parent` ref so no path strings to update. Concurrency: acquire locks on both parents — use canonical ordering (by hash) to prevent deadlock.

---

## Concurrency Probe (~40 min)

> "Two processes write to the same file simultaneously. What happens?"

Strong: `File.content` is a `StringBuilder` — not thread-safe. Concurrent `append()` calls corrupt content. Use `synchronized` on `File.write()` or use `StringBuffer`. For directories: `ReentrantReadWriteLock` allows concurrent `ls`/`size` (reads) with exclusive `addChild`/`removeChild` (writes).

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | `ls` duality, `mkdir` idempotency, relative path support, `find` | |
| Entity Modeling | `FileSystemComponent` base, `Map` for children, `parent` ref | |
| Design Patterns | Composite (uniform `size()` on leaf and composite) | |
| SOLID | OCP: new component type = new subclass; no `instanceof` in client | |
| Extensibility | Permissions as Proxy; symlinks as new component type | |
| Concurrency | `ReentrantReadWriteLock` per Directory; `synchronized` on File write | |
| Communication | Explained Composite insight, WHY `Map` over `List`, WHY `parent` ref | |
| **TOTAL** | | **/21** |
