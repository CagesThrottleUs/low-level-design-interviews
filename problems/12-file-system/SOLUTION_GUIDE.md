# Solution Guide — In-Memory File System

**Read after your attempt.**

---

## Entity Map

| Class | Type | Responsibility |
|-------|------|---------------|
| `FileSystemComponent` | Abstract (Composite base) | `name`, `parent` ref, timestamps; declares `size()`, `isDirectory()`, `print()` |
| `File` | Concrete Leaf | `StringBuilder content`; `size()` = content length |
| `Directory` | Concrete Composite | `Map<String, FileSystemComponent>` children; delegates `size()` recursively; `ls()`, `find()` |
| `FileSystem` | Singleton Facade | Owns `root` and `currentDirectory`; all path-resolution logic |
| `FileSystemException` | RuntimeException | Invalid path, type mismatch |

---

## Composite Pattern — The Core Insight

```java
// Client calls size() without knowing if it's a file or directory
FileSystemComponent entry = filesystem.resolve("/home");
int totalSize = entry.size(); // works uniformly on both

// Without Composite: ugly instanceof checks everywhere
if (entry instanceof File) return ((File)entry).getContent().length();
else if (entry instanceof Directory) { /* recursive logic */ }
```

---

## Full Implementation

```java
public abstract class FileSystemComponent {
    protected final String name;
    protected FileSystemComponent parent;
    protected long createdAt = System.currentTimeMillis();

    public FileSystemComponent(String name, FileSystemComponent parent) {
        this.name = name; this.parent = parent;
    }

    public String getName() { return name; }
    public FileSystemComponent getParent() { return parent; }
    void setParent(FileSystemComponent p) { this.parent = p; }

    public abstract int size();
    public abstract boolean isDirectory();
    public abstract void print(String indent);
}

public class File extends FileSystemComponent {
    private StringBuilder content = new StringBuilder();

    public File(String name, FileSystemComponent parent) {
        super(name, parent);
    }

    @Override public int size()          { return content.length(); }
    @Override public boolean isDirectory() { return false; }

    public synchronized void write(String text, boolean append) {
        if (!append) content = new StringBuilder();
        content.append(text);
    }

    public String getContent() { return content.toString(); }

    @Override public void print(String indent) {
        System.out.println(indent + name + " (" + size() + "B)");
    }
}

public class Directory extends FileSystemComponent {
    private final Map<String, FileSystemComponent> children = new LinkedHashMap<>();
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public Directory(String name, FileSystemComponent parent) {
        super(name, parent);
    }

    @Override public boolean isDirectory() { return true; }

    // Composite: recursively sums children — uniform with File.size()
    @Override
    public int size() {
        lock.readLock().lock();
        try {
            int total = 0;
            for (FileSystemComponent c : children.values()) total += c.size();
            return total;
        } finally { lock.readLock().unlock(); }
    }

    public void addChild(FileSystemComponent c) {
        lock.writeLock().lock();
        try {
            children.put(c.getName(), c);
            c.setParent(this);
        } finally { lock.writeLock().unlock(); }
    }

    public boolean removeChild(String name) {
        lock.writeLock().lock();
        try { return children.remove(name) != null; }
        finally { lock.writeLock().unlock(); }
    }

    public FileSystemComponent getChild(String name) {
        lock.readLock().lock();
        try { return children.get(name); }
        finally { lock.readLock().unlock(); }
    }

    public boolean hasChild(String name) {
        lock.readLock().lock();
        try { return children.containsKey(name); }
        finally { lock.readLock().unlock(); }
    }

    public List<String> ls() {
        lock.readLock().lock();
        try {
            List<String> names = new ArrayList<>(children.keySet());
            Collections.sort(names);
            return names;
        } finally { lock.readLock().unlock(); }
    }

    public List<FileSystemComponent> find(String targetName) {
        List<FileSystemComponent> results = new ArrayList<>();
        findRecursive(this, targetName, results);
        return results;
    }

    private void findRecursive(Directory dir, String name, List<FileSystemComponent> out) {
        lock.readLock().lock();
        try {
            for (FileSystemComponent c : dir.children.values()) {
                if (c.getName().equals(name)) out.add(c);
                if (c.isDirectory()) findRecursive((Directory) c, name, out);
            }
        } finally { lock.readLock().unlock(); }
    }

    @Override public void print(String indent) {
        System.out.println(indent + "[" + name + "]");
        children.values().forEach(c -> c.print(indent + "  "));
    }
}
```

---

## FileSystem Facade (Path Resolution)

```java
public class FileSystem {
    private static volatile FileSystem instance;
    private final Directory root = new Directory("/", null);
    private Directory currentDir = root;

    public static FileSystem getInstance() {
        if (instance == null) synchronized (FileSystem.class) {
            if (instance == null) instance = new FileSystem();
        }
        return instance;
    }

    private Directory resolveDirPath(String path, boolean createMissing) {
        String[] parts = path.split("/");
        Directory cur = path.startsWith("/") ? root : currentDir;
        for (String part : parts) {
            if (part.isEmpty() || part.equals(".")) continue;
            if (part.equals("..")) {
                if (cur.getParent() != null) cur = (Directory) cur.getParent();
                continue;
            }
            FileSystemComponent child = cur.getChild(part);
            if (child == null) {
                if (!createMissing) throw new FileSystemException("No such directory: " + part);
                Directory newDir = new Directory(part, cur);
                cur.addChild(newDir);
                child = newDir;
            }
            if (!child.isDirectory()) throw new FileSystemException("Not a directory: " + part);
            cur = (Directory) child;
        }
        return cur;
    }

    public void mkdir(String path) { resolveDirPath(path, true); }

    public void touch(String path) {
        int i = path.lastIndexOf('/');
        Directory dir = resolveDirPath(i <= 0 ? "/" : path.substring(0, i), true);
        String name = path.substring(i + 1);
        if (!dir.hasChild(name)) dir.addChild(new File(name, dir));
    }

    public List<String> ls(String path) {
        int i = path.lastIndexOf('/');
        Directory parent = resolveDirPath(i <= 0 ? "/" : path.substring(0, i), false);
        String lastName = path.substring(i + 1);
        if (lastName.isEmpty()) return parent.ls();
        FileSystemComponent target = parent.getChild(lastName);
        if (target == null) throw new FileSystemException("No such entry: " + path);
        return target.isDirectory() ? ((Directory) target).ls()
                                    : List.of(target.getName()); // ls on file
    }

    public String readFile(String path) {
        int i = path.lastIndexOf('/');
        Directory dir = resolveDirPath(i <= 0 ? "/" : path.substring(0, i), false);
        FileSystemComponent f = dir.getChild(path.substring(i + 1));
        if (f == null || f.isDirectory()) throw new FileSystemException("Not a file: " + path);
        return ((File) f).getContent();
    }

    public void writeFile(String path, String content, boolean append) {
        int i = path.lastIndexOf('/');
        Directory dir = resolveDirPath(i <= 0 ? "/" : path.substring(0, i), true);
        String name = path.substring(i + 1);
        FileSystemComponent entry = dir.getChild(name);
        if (entry == null) {
            File file = new File(name, dir);
            file.write(content, false);
            dir.addChild(file);
        } else if (!entry.isDirectory()) {
            ((File) entry).write(content, append);
        } else throw new FileSystemException("Is a directory: " + name);
    }

    public void rm(String path) {
        int i = path.lastIndexOf('/');
        Directory dir = resolveDirPath(i <= 0 ? "/" : path.substring(0, i), false);
        if (!dir.removeChild(path.substring(i + 1)))
            throw new FileSystemException("Not found: " + path);
    }

    public void cd(String path) { currentDir = resolveDirPath(path, false); }

    public String pwd() {
        if (currentDir == root) return "/";
        Deque<String> parts = new ArrayDeque<>();
        FileSystemComponent node = currentDir;
        while (node.getParent() != null) { parts.push(node.getName()); node = node.getParent(); }
        StringBuilder sb = new StringBuilder();
        while (!parts.isEmpty()) sb.append("/").append(parts.pop());
        return sb.toString();
    }

    public List<FileSystemComponent> find(String path, String name) {
        return resolveDirPath(path, false).find(name);
    }

    public int size(String path) {
        int i = path.lastIndexOf('/');
        Directory dir = resolveDirPath(i <= 0 ? "/" : path.substring(0, i), false);
        String last = path.substring(i + 1);
        FileSystemComponent target = last.isEmpty() ? dir : dir.getChild(last);
        if (target == null) throw new FileSystemException("Not found: " + path);
        return target.size();
    }
}
```

---

## Extension Points

| "Now add..." | Change |
|-------------|--------|
| Permissions | `int permissions` bitmask on component; check in FileSystem before ops |
| Symlinks | `Symlink extends FileSystemComponent` with `target` ref; path resolution detects cycles |
| `mv` | Remove from src parent, re-parent to dst; use lock ordering on both directories |
| `cp` | Deep clone via Visitor or recursive `clone()` |
| Search by extension | `SearchStrategy` interface — Strategy pattern |

---

## What Strong Candidates Do Differently

- `Map<String, FileSystemComponent>` not `List` — O(1) child lookup
- `parent` ref in each node — `pwd()` reconstructs path without storing strings
- `ls` returns `[filename]` for file paths — matches real `ls` semantics
- `mkdir` is idempotent — no error if directory exists
- `split("/")` empty-part handling: `if (part.isEmpty()) continue`
- `ReentrantReadWriteLock` per Directory — concurrent reads, exclusive writes

## What Average Candidates Miss

- `List` for children — O(n) per path segment
- Storing full path string in each node — breaks on `mv`
- `ls` throws on file path instead of returning `[filename]`
- `mkdir` throws if directory exists (non-idempotent)
- Forgetting `split("/")` produces empty strings on absolute paths
