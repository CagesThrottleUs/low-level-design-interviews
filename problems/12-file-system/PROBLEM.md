# In-Memory File System

**Difficulty:** Intermediate
**Category:** Composite Pattern + Recursive Tree
**Time Box:** 45 min (discussion) / 90 min (machine coding)
**Key Patterns:** Composite (File and Directory as unified component), Proxy (access control extension)
**Asked at:** Google, Amazon, Microsoft, LeetCode #588

---

## Problem Statement

Design an in-memory file system that supports creating files and directories, reading and writing file content, listing directory contents, and navigating paths. Files and directories should be treated uniformly where possible — for example, `size()` works on both a file (returns its content length) and a directory (returns the sum of all children's sizes recursively).

This is the canonical Composite pattern problem: leaves (files) and composites (directories) share a common interface.

---

## Actors

| Actor | Description |
|-------|-------------|
| User / Process | Executes file system commands (mkdir, touch, ls, cat, etc.) |
| FileSystem | Manages the root directory tree; resolves paths; dispatches operations |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: `mkdir(path)` — create directory and all missing parents (like `mkdir -p`)
- FR2: `touch(path)` — create an empty file at the given path
- FR3: `ls(path)` — if path is a file, return `[filename]`; if directory, return sorted list of children
- FR4: `cat(path)` / `readFile(path)` — return file contents as string
- FR5: `echo(path, content, append)` / `writeFile(path, content, append)` — write or append to file
- FR6: `rm(path)` — delete a file or directory (recursively)
- FR7: `pwd()` — return current working directory path
- FR8: `cd(path)` — change current directory; support `..`
- FR9: `size(path)` — file: content length; directory: recursive sum of all file sizes
- FR10: `find(path, name)` — recursive DFS search by name

**Secondary (implement if time allows):**
- FR11: `mv(src, dst)` — move file or directory
- FR12: `cp(src, dst)` — copy (deep clone for directory)
- FR13: Permissions per entry (read/write/execute per user/group/other)
- FR14: Symbolic links

---

## Non-Functional Requirements

- **Correctness:** `ls` on a file path returns `[filename]`, not an error
- **Path handling:** Support both absolute (`/home/user/file`) and relative paths (`../sibling`)
- **Thread safety:** Multiple processes can read/write simultaneously
- **Idempotency:** `mkdir` on an existing directory is a no-op (not an error)

---

## Constraints and Assumptions

- In-memory only — no persistence
- Paths use `/` as separator
- Root directory is `/`
- `split("/")` on `/home/user` produces `["", "home", "user"]` — empty strings must be skipped
- Out of scope: disk quotas, hard links, symlinks (unless time allows)

---

## Good Clarifying Questions to Ask

1. Do we need absolute and relative path support, or only absolute?
2. Should `ls` on a file path return an error or return `[filename]`?
3. Should `mkdir` be idempotent (no error if directory already exists)?
4. Do we need recursive delete (`rm -rf`) or only empty directories?
5. Is thread safety required — multiple concurrent reads/writes?
6. Should `size()` count only file content or include directory metadata?

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **FileSystemComponent** — abstract base; has `name`, `parent` ref, timestamps; declares `size()`, `isDirectory()`, `print(indent)`
- **File** (Leaf) — extends component; holds `StringBuilder content`; `size()` = content length
- **Directory** (Composite) — extends component; holds `Map<String, FileSystemComponent>`; `size()` recursively sums children; `ls()` returns sorted keys; `find()` does DFS
- **FileSystem** — Singleton Facade; holds `root: Directory` and `currentDirectory`; owns all path-resolution logic
- **FileSystemException** — RuntimeException for invalid paths, type mismatches

The key Composite insight: `component.size()` works uniformly on both File and Directory without the caller knowing which type it is.

</details>
