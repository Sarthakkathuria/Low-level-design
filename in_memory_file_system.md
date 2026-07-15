# In-Memory File System

An in-memory file system mirrors the familiar hierarchy of folders and files — single root, nested directories, files with content — except everything lives in RAM. No disk, no I/O. That lets us focus purely on the data structure and the operations, which is exactly where the design challenge lies.

---

## Requirements

You walk into the interview and get greeted with this prompt:

> "Design an in-memory file system that supports creating files and directories, navigating paths, and basic file operations."

Before touching a whiteboard, spend 3–5 minutes narrowing that down.

### Clarifying Questions

**You:** "Is this a single-rooted hierarchy, like a Unix filesystem starting at `/`?"
**Interviewer:** "Yes. Single root, absolute paths only."

**You:** "What do files store — just strings, or arbitrary bytes?"
**Interviewer:** "String content is fine."

**You:** "What operations do we need beyond create and delete? Read, write, list?"
**Interviewer:** "Yes, plus move and rename."

**You:** "When I move a folder, does its entire subtree move with it?"
**Interviewer:** "Yes."

**You:** "Do we need to support relative paths like `../` or `./`?"
**Interviewer:** "No, absolute paths only."

**You:** "What about permissions, timestamps, or symbolic links?"
**Interviewer:** "All out of scope. Focus on structure and operations."

**You:** "How large can this get? Tens of files or tens of thousands?"
**Interviewer:** "Tens of thousands of entries should be fine in memory."

That last answer matters. It rules out naive O(n) scans across all entries and points us toward O(1) child lookups — a `Map` for children rather than a `List`.

### Final Requirements

| # | Requirement |
|---|-------------|
| 1 | Hierarchical file system with a single root directory |
| 2 | Files store string content |
| 3 | Folders contain files and other folders |
| 4 | Create and delete files and folders |
| 5 | List contents of a folder |
| 6 | Navigate/resolve absolute paths (e.g., `/home/user/docs`) |
| 7 | Rename and move files and folders |
| 8 | Retrieve full path from any file/folder reference |
| 9 | Scale to tens of thousands of entries in memory |

**Out of scope:** search, relative paths (`../`, `./`), permissions, timestamps, symbolic links, persistence, UI layer.

---

## Core Entities and Relationships

Both files and directories share common properties: a name, a parent, and the ability to report their own full path. The difference is that directories hold children and files hold content. This is the classic **Composite** structure — a natural fit here.

**Node** — The shared abstraction for both files and directories. Owns the name, the parent reference, and the logic to construct a full path by walking up the tree. Without a base class, we'd duplicate name/parent logic in both File and Directory.

**File** — A leaf in the tree. Stores string content and exposes read/write. No children.

**Directory** — An internal node in the tree. Holds a `Map<String, Node>` of named children. Enforces uniqueness of names within a directory.

**FileSystem** — The orchestrator. Holds the root directory and exposes the public API. All path resolution and operation logic lives here. Callers never manipulate nodes directly.

### Entity Responsibilities

| Entity | Responsibility |
|--------|---------------|
| `FileSystem` | Entry point for all operations. Resolves paths, delegates to nodes, enforces invariants like cycle prevention on move. |
| `Directory` | Internal node. Owns a named child map. Manages add/remove of children. |
| `File` | Leaf node. Stores and exposes string content. |
| `Node` | Abstract base. Owns name and parent reference. Provides `getFullPath()` by walking up the tree. |

### Relationships

```
FileSystem
  └── root: Directory

Node (abstract base)
  ├── name: String
  └── parent: Directory?      <-- null only for root

Directory extends Node
  └── children: Map<String, Node>
         ├── "subdir" -> Directory
         └── "file.txt" -> File

File extends Node
  └── content: String
```

---

## Class Design

We design top-down: FileSystem first (the entry point), then Directory, then File, then the Node base.

### FileSystem

FileSystem is what callers interact with. It resolves paths, performs operations, and keeps the tree consistent.

**Deriving State**

| Requirement | What FileSystem must track |
|-------------|---------------------------|
| "Single root directory" | A reference to the root Directory |

That's it. FileSystem doesn't need a flat registry of all nodes. The tree is self-indexing — navigate from root to reach any node.

**Deriving Behavior**

| Need from requirements | Method on FileSystem |
|------------------------|----------------------|
| "Create files" | `createFile(path, content) -> File` |
| "Create folders" | `createDirectory(path) -> Directory` |
| "Delete files and folders" | `delete(path) -> void` |
| "List contents of a folder" | `list(path) -> List<String>` |
| "Move files and folders" | `move(sourcePath, destPath) -> void` |
| "Rename files and folders" | `rename(path, newName) -> void` |
| "Retrieve file content" | `readFile(path) -> String` |
| "Write file content" | `writeFile(path, content) -> void` |

```
class FileSystem:
    - root: Directory

    + FileSystem()
    + createFile(path, content) -> File
    + createDirectory(path) -> Directory
    + delete(path) -> void
    + list(path) -> List<String>
    + move(sourcePath, destPath) -> void
    + rename(path, newName) -> void
    + readFile(path) -> String
    + writeFile(path, content) -> void
```

`resolvePath(path) -> Node` is a private helper. It's used by every public method internally but not exposed — callers work with paths, not node references. This keeps the API clean and prevents callers from holding stale references after a rename or delete.

---

### Node (abstract base)

Node captures everything shared between File and Directory.

**Deriving State**

| Requirement | What Node must track |
|-------------|----------------------|
| "Navigate/resolve paths" | Name of this entry |
| "Move and rename" | Parent reference (needed to detach from old parent) |
| "Retrieve full path from any reference" | Parent reference (walk up to build path) |

**Deriving Behavior**

| Need | Method on Node |
|------|---------------|
| "Rename files/folders" | `getName()`, `setName(name)` |
| "Retrieve full path" | `getFullPath() -> String` |
| "Move: detach from parent" | `getParent()`, `setParent(dir)` |

```
abstract class Node:
    - name: String
    - parent: Directory?     // null only for root

    + getName() -> String
    + setName(name) -> void
    + getParent() -> Directory?
    + setParent(dir) -> void
    + getFullPath() -> String
```

`getFullPath()` walks up the parent chain to reconstruct the absolute path. This satisfies requirement 8 without FileSystem being involved — the node knows how to describe itself.

---

### Directory extends Node

Directory is the internal node. Its core data structure choice — `Map<String, Node>` — is the key decision at scale.

**Deriving State**

| Requirement | What Directory must track |
|-------------|--------------------------|
| "Folders contain files and other folders" | A named map of children |
| "Scale to tens of thousands of entries" | O(1) lookup by name (Map, not List) |

Using `Map<String, Node>` gives O(1) lookup by child name. Path resolution navigates one segment at a time, so each step is O(1) rather than O(n) — essential at tens of thousands of entries.

**Deriving Behavior**

| Need | Method on Directory |
|------|---------------------|
| "Navigate paths: find child by name" | `getChild(name) -> Node?` |
| "List contents" | `listChildren() -> List<String>` |
| "Create: add child" | `addChild(node) -> void` |
| "Delete/move: remove child" | `removeChild(name) -> void` |
| "Prevent name collisions" | `hasChild(name) -> boolean` |

```
class Directory extends Node:
    - children: Map<String, Node>

    + Directory(name, parent)
    + getChild(name) -> Node?
    + hasChild(name) -> boolean
    + addChild(node) -> void
    + removeChild(name) -> void
    + listChildren() -> List<String>
```

---

### File extends Node

File is the leaf. Its only unique responsibility is storing and exposing content.

**Deriving State**

| Requirement | What File must track |
|-------------|----------------------|
| "Files store string content" | The content string |

**Deriving Behavior**

| Need | Method on File |
|------|---------------|
| "Read file content" | `read() -> String` |
| "Write file content" | `write(content) -> void` |

```
class File extends Node:
    - content: String

    + File(name, parent)
    + read() -> String
    + write(content) -> void
```

---

### Final Class Design

```
+-----------------------+
|      FileSystem       |
+-----------------------+
| - root: Directory     |
+-----------------------+
| + createFile()        |
| + createDirectory()   |
| + delete()            |
| + list()              |
| + move()              |
| + rename()            |
| + readFile()          |
| + writeFile()         |
+-----------------------+
           |
           | owns root
           v
+---------------------+          (abstract)
|    Node             |<--------------------------+
+---------------------+                           |
| - name: String      |                           |
| - parent: Directory |                    extends |
+---------------------+                           |
| + getName()         |          +----------------+-------+
| + setName()         |          |                        |
| + getParent()       |    +-----+--------+    +----------+------+
| + setParent()       |    |   Directory  |    |     File         |
| + getFullPath()     |    +--------------+    +------------------+
+---------------------+    | - children:  |    | - content: String|
                           |   Map<String,|    +------------------+
                           |   Node>      |    | + read()         |
                           +--------------+    | + write()        |
                           | + getChild() |    +------------------+
                           | + hasChild() |
                           | + addChild() |
                           | + removeChild|
                           | + listChildren|
                           +--------------+
```

---

## Implementation

The two most interesting methods are `resolvePath()` (the engine behind every operation) and `move()` (the most nuanced operation due to cycle prevention). We'll also walk through `createFile()` and `delete()`.

### resolvePath() — private helper

Every public method calls this first. It walks the path segment by segment from root.

**Core logic:**
1. If path is `/`, return root
2. Split on `/`, discard empty segments
3. Walk the tree one segment at a time
4. Return the node at the end, or null if not found

**Edge cases:**
- Path is `/` → return root
- Segment not found → return null
- Trying to descend into a File → return null (files have no children)

```
resolvePath(path)
    if path == "/"
        return root

    parts = path.split("/").filter(not empty)
    current = root

    for part in parts
        if !(current instanceof Directory)
            return null    // can't cd into a file
        current = current.getChild(part)
        if current == null
            return null

    return current
```

---

### createFile(path, content)

**Core logic:**
1. Split path into parent path and file name
2. Resolve the parent directory
3. Check no name conflict
4. Create File, add to parent

**Edge cases:**
- Parent path doesn't exist
- Parent path resolves to a File, not a Directory
- Name already exists at that location
- File name is empty

```
createFile(path, content)
    parentPath = path.substring(0, path.lastIndexOf("/")) || "/"
    fileName   = path.substring(path.lastIndexOf("/") + 1)

    if fileName.isEmpty()
        throw Error("Invalid path")

    parentDir = resolvePath(parentPath)
    if parentDir == null || !(parentDir instanceof Directory)
        throw Error("Parent directory not found: " + parentPath)

    if parentDir.hasChild(fileName)
        throw Error("Already exists: " + fileName)

    file = File(fileName, parentDir)
    file.write(content)
    parentDir.addChild(file)
    return file
```

`createDirectory(path)` follows the same pattern — resolve parent, check no conflict, create Directory, add to parent. The only difference is the node type created.

---

### delete(path)

**Core logic:**
1. Resolve the node
2. Detach it from its parent
3. Done — subtree is garbage collected automatically

**Edge cases:**
- Path not found
- Attempting to delete root

```
delete(path)
    node = resolvePath(path)
    if node == null
        throw Error("Path not found: " + path)
    if node.getParent() == null
        throw Error("Cannot delete root")

    node.getParent().removeChild(node.getName())
```

There is no recursive cleanup needed. Once we remove the directory from its parent's map, nothing else holds a reference to it. The entire subtree — every child File and Directory — becomes unreachable and is garbage collected. This is one of the advantages of a reference-based tree over a flat registry.

---

### move(sourcePath, destPath)

Move is the most nuanced operation. The critical edge case is **cycle prevention** — moving a directory into one of its own descendants would make the tree unreachable.

**Core logic:**
1. Resolve source and destination
2. Validate both exist
3. Detect cycles: walk up from dest, check if source is an ancestor
4. Detach source from its current parent
5. Attach source to destination

**Edge cases:**
- Source not found
- Destination not found, or not a Directory
- Attempting to move root
- Destination is inside source (cycle)
- Name conflict at destination

```
move(sourcePath, destPath)
    source = resolvePath(sourcePath)
    if source == null
        throw Error("Source not found")
    if source.getParent() == null
        throw Error("Cannot move root")

    dest = resolvePath(destPath)
    if dest == null || !(dest instanceof Directory)
        throw Error("Destination directory not found")

    if dest.hasChild(source.getName())
        throw Error("Name conflict at destination")

    // Cycle check: dest must not be inside source
    current = dest
    while current != null
        if current == source
            throw Error("Cannot move directory into its own subdirectory")
        current = current.getParent()

    // Perform move
    source.getParent().removeChild(source.getName())
    dest.addChild(source)
    source.setParent(dest)
```

The cycle check walks upward from `dest`. If it ever reaches `source`, the move would create a cycle. This is O(depth) — acceptable since tree depth is bounded in practice.

---

### getFullPath() on Node

`getFullPath()` satisfies requirement 8 without FileSystem being involved. The node walks its own parent chain.

```
getFullPath()
    if parent == null
        return "/"
    parentPath = parent.getFullPath()
    return (parentPath == "/" ? "" : parentPath) + "/" + name
```

A file at `/home/user/notes.txt` reconstructs its path as: `notes.txt` asks parent `user`, which asks parent `home`, which asks parent root → `"/"`. Unwinding: `"/home"` → `"/home/user"` → `"/home/user/notes.txt"`.

---

## Design Patterns

### Composite Pattern
Directory and File share a common base class (Node) with different structure: Directory holds other Nodes, File is a leaf. This is the Composite pattern — a tree where internal nodes and leaves share an interface and internal nodes can contain either type. It's the right pattern for any recursive containment hierarchy (file systems, UI component trees, org charts).

The key benefit: `delete()` and `move()` operate on any Node without caring whether it's a File or Directory. `resolvePath()` navigates by asking each Directory for a child — it doesn't know or care what type that child is until it needs to descend further.

### Information Expert
Each class owns the logic that uses its own data:
- `Node` builds its own full path (owns `name` and `parent`)
- `Directory` enforces child uniqueness (owns `children` map)
- `File` manages its own content (owns `content`)
- `FileSystem` validates paths and enforces cross-cutting rules like cycle prevention (owns the root and sees the full tree)

---

## Verification

Let's trace a sequence of operations to verify state transitions.

**Initial state:** root `/` is an empty Directory.

**Create `/home`:**
```
createDirectory("/home")
  parentPath = "/"  ->  resolvePath("/") = root
  root.hasChild("home") -> false
  dir = Directory("home", root)
  root.addChild(dir)
  State: root.children = {"home" -> Directory}
```

**Create `/home/user`:**
```
createDirectory("/home/user")
  parentPath = "/home"  ->  resolvePath("/home") = home
  home.hasChild("user") -> false
  dir = Directory("user", home)
  home.addChild(dir)
```

**Create `/home/user/notes.txt`:**
```
createFile("/home/user/notes.txt", "Hello World")
  parentPath = "/home/user"  ->  user directory
  user.hasChild("notes.txt") -> false
  file = File("notes.txt", user)
  file.write("Hello World")
  user.addChild(file)
```

**Read the file:**
```
readFile("/home/user/notes.txt")
  node = resolvePath("/home/user/notes.txt") -> File
  return file.read()  ->  "Hello World"
```

**Get full path from node reference:**
```
file.getFullPath()
  parent = user  ->  user.getFullPath()
    parent = home  ->  home.getFullPath()
      parent = root  ->  return "/"
    return "/home"
  return "/home/user"
return "/home/user/notes.txt"
```

**Move `/home/user/notes.txt` to `/home`:**
```
move("/home/user/notes.txt", "/home")
  source = File("notes.txt")
  dest   = Directory("home")
  dest.hasChild("notes.txt") -> false
  Cycle check: walk up from home -> root -> null. Source not found. Safe.
  user.removeChild("notes.txt")
  home.addChild(file)
  file.setParent(home)

  State: user.children={}, home.children={"user", "notes.txt"}
```

**List `/home`:**
```
list("/home")
  dir = resolvePath("/home") -> home
  home.listChildren() -> ["user", "notes.txt"]
```

**Attempt to move `/home` into `/home/user` (cycle):**
```
move("/home", "/home/user")
  source = Directory("home")
  dest   = Directory("user")
  Cycle check: walk up from user:
    user.getParent() = home = source  ->  MATCH
  throw Error("Cannot move directory into its own subdirectory")
  State unchanged.
```

---

## Extensibility

### 1. "How would you add relative path support (`../`, `./`)?"

Right now `resolvePath()` always starts from root. To support relative paths, it needs a current working directory context.

> "I'd add a `currentDirectory: Directory` field to FileSystem. When a path starts with `/`, resolve from root as today. When it starts with `./` or doesn't start with `/`, resolve from `currentDirectory`. For `../`, each `..` segment calls `getParent()` instead of `getChild()`. The tree structure doesn't change at all — only the entry point for path resolution changes."

```
class FileSystem:
    - root: Directory
    - currentDirectory: Directory   // NEW

resolvePath(path)
    start = path.startsWith("/") ? root : currentDirectory
    parts = normalizePath(path)     // handle ./ and ../
    // walk from start
```

---

### 2. "How would you support permissions or ownership?"

> "I'd add `owner` and `permissions` fields to `Node` — since all nodes (both File and Directory) need them, the base class is the right place. A separate `PermissionChecker` could enforce access rules before any FileSystem operation executes. None of the existing operation logic changes — we'd add guard clauses at the top of each public method: `permissionChecker.checkRead(node, currentUser)`. This keeps authorization separate from the file system logic itself."

---

### 3. "How would you add search across the file system?"

Right now there's no way to find a file by name without knowing its path. Adding search is purely additive.

> "I'd add a `search(name) -> List<Node>` method to FileSystem that does a DFS from root, collecting nodes whose name matches. The tree structure already supports traversal — Directory gives us `listChildren()` to iterate. No existing class changes. If search needed to be O(1), I'd add a `Map<String, List<Node>>` index on FileSystem that's updated on every create, delete, and rename. That's the trade-off: memory overhead for fast lookup."

---

### 4. "How would you make this file system thread-safe?"

The core problem is that multiple threads can interleave reads and writes on the same tree. A `resolvePath` walking down the tree while another thread is executing a `move` or `delete` on the same path can produce inconsistent results or null pointer errors.

There are two practical approaches, and the right answer is to start simple and explain when you'd step up.

**Option 1: Single global ReadWriteLock (start here)**

> "I'd add a `ReadWriteLock` to FileSystem. Operations that don't modify the tree — `list`, `readFile`, `resolvePath` — acquire the read lock. Multiple threads can hold the read lock simultaneously. Operations that modify the tree — `createFile`, `createDirectory`, `delete`, `move`, `rename`, `writeFile` — acquire the write lock exclusively. This is simple to implement and correct for most workloads. The trade-off is that a long write blocks all reads, but for a file system where reads dominate, that's often acceptable."

```
class FileSystem:
    - root: Directory
    - lock: ReadWriteLock

readFile(path)
    lock.readLock().acquire()
    try
        node = resolvePath(path)
        return node.read()
    finally
        lock.readLock().release()

createFile(path, content)
    lock.writeLock().acquire()
    try
        // ... existing logic
    finally
        lock.writeLock().release()
```

**Option 2: Per-directory locking (if interviewer pushes for more concurrency)**

> "If we need two threads to modify `/home/a` and `/home/b` concurrently without blocking each other, we'd give each Directory its own ReadWriteLock. This is fine-grained locking — operations on unrelated subtrees don't block each other. The tricky case is `move()`, which touches two directories. To avoid deadlock, we always acquire locks in a consistent order — for example, by the node's depth in the tree, or by a stable ID assigned at creation. Never acquire the deeper lock first. Without a consistent ordering, two concurrent moves in opposite directions can deadlock on each other."

```
class Directory extends Node:
    - children: Map<String, Node>
    - lock: ReadWriteLock        // NEW: per-directory lock

move(sourcePath, destPath)
    source = resolvePath(sourcePath)
    dest   = resolvePath(destPath)

    // Always lock the shallower node first to prevent deadlock
    first, second = orderByDepth(source.getParent(), dest)
    first.lock.writeLock().acquire()
    second.lock.writeLock().acquire()
    try
        // ... existing move logic
    finally
        second.lock.writeLock().release()
        first.lock.writeLock().release()
```

For the interview scope, the global `ReadWriteLock` is the right answer. It's simple, demonstrably correct, and shows you understand the read/write distinction. Mention per-directory locking as the next step if the interviewer asks what you'd do for higher concurrency, but don't implement it upfront.
