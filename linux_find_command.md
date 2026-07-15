# Linux Find Command API

A file search API that replicates the Linux `find` command. Users specify a root directory and one or more filters — by name, extension, or size — and the system recursively traverses the directory tree, returning every file that satisfies all conditions.

---

## Requirements

You walk into the interview and get greeted with this prompt:

> "Design an API service that replicates the Linux 'find' command, allowing users to search files across directories based on criteria like size, format, and name. The system should support filters like 'files >5MB' or 'all XML files' through a clean, extensible interface."

### Clarifying Questions

**You:** "What filter types do we need to support?"
**Interviewer:** "Size comparisons — greater than and less than — file extension, and name matching. That covers the core use cases."

**You:** "Can users combine multiple filters? Like 'all XML files greater than 5MB'?"
**Interviewer:** "Yes. When multiple filters are provided, all conditions must match."

**You:** "Are we operating on the real OS file system or an in-memory abstraction?"
**Interviewer:** "Model it as an in-memory file system. No actual OS calls."

**You:** "What does the result look like — just file paths, or richer metadata?"
**Interviewer:** "Return file objects with name, size, extension, and full path."

**You:** "Should search be recursive into subdirectories by default?"
**Interviewer:** "Yes, fully recursive."

**You:** "Should directories appear in results, or files only?"
**Interviewer:** "Files only."

**You:** "Case sensitivity for name and extension matching?"
**Interviewer:** "Case-insensitive."

**You:** "Concurrency?"
**Interviewer:** "Skip it for now."

### Final Requirements

| # | Requirement |
|---|-------------|
| 1 | Users specify a root directory path and one or more filters |
| 2 | Supported filters: name (case-insensitive substring), extension (e.g., "xml" or ".xml"), size greater than N bytes, size less than N bytes |
| 3 | Multiple filters compose with AND logic — all conditions must match |
| 4 | Search is recursive through all subdirectories |
| 5 | Returns a list of matching File objects (name, size, extension, path) |
| 6 | Results include files only — directories are not returned |

**Out of scope:** real OS file system calls, OR/NOT logical composition, hidden files, file permissions, date/timestamp filters, file content search, concurrency.

---

## Core Entities and Relationships

The central challenge here is **filter extensibility**. The prompt explicitly asks for a "clean, extensible interface." A hard-coded if-else chain (`if type == "size" … else if type == "extension" …`) breaks every time a new filter type is added, and combining filters gets messy fast. This points directly to the **Strategy pattern** — each filter is an independent predicate object with a single `apply(file) -> boolean` method. Combining them is handled either by the caller passing a list (AND semantics) or by a **Composite filter** that wraps a list and applies OR or AND logic explicitly.

**FileSearchAPI** — The orchestrator and public entry point. Accepts a root directory path and a list of filters, traverses the directory tree recursively, applies all filters to each file, and returns matches. Maintains a flat registry of directories by path for O(1) lookup — without it, finding a deep subdirectory by path would require a full tree walk on every search call.

**Directory** — A node in the in-memory file system. Owns a list of child Files and a list of child Directories. No traversal logic lives here — Directory is a pure data container. The traversal is driven top-down by FileSearchAPI.

**File** — A leaf node. Stores name, extension, size in bytes, and full path. Immutable after creation. The unit that all filters operate on.

**Filter (interface)** — The strategy abstraction. One method: `apply(file) -> boolean`. Every filter type implements this. The entire extensibility story lives here — adding a new filter means implementing this interface and nothing else.

**MinSizeFilter** — Passes files strictly larger than a given byte threshold.

**MaxSizeFilter** — Passes files strictly smaller than a given byte threshold.

**ExtensionFilter** — Passes files whose extension matches (case-insensitive). Normalizes the input at construction so `apply()` is a plain equality check.

**NameFilter** — Passes files whose name contains a given substring (case-insensitive).

**AndFilter** — Composite filter. Takes a list of filters and passes a file only if every child filter passes. Useful when the caller wants to pre-build a reusable composed filter as a single object — particularly valuable once `OrFilter` exists and you need to mix AND/OR logic in a tree.

### Entity Responsibilities

| Entity | Responsibility |
|--------|----------------|
| `FileSearchAPI` | Entry point. Holds the in-memory file system. Routes search calls into recursive traversal. |
| `Directory` | Owns child files and subdirectories. Pure data — no traversal logic. |
| `File` | Stores file metadata. The unit filters operate on. |
| `Filter` | Interface. Single predicate: does this file match? |
| `MinSizeFilter` | Concrete filter: size > threshold. |
| `MaxSizeFilter` | Concrete filter: size < threshold. |
| `ExtensionFilter` | Concrete filter: extension matches (normalized, case-insensitive). |
| `NameFilter` | Concrete filter: name contains substring (case-insensitive). |
| `AndFilter` | Composite filter: delegates to all children, passes only if all pass. |

### Relationships

```
FileSearchAPI
  ├── directories: Map<path, Directory>   ← flat registry for O(1) lookup
  └── search(rootPath, filters) -> List<File>

Directory
  ├── name: String
  ├── path: String
  ├── files: List<File>
  └── subdirectories: List<Directory>

File
  ├── name: String
  ├── extension: String
  ├── size: long
  └── path: String

<<interface>> Filter
  └── apply(file: File) -> boolean
        ^             ^              ^           ^           ^
        |             |              |           |           |
  MinSizeFilter MaxSizeFilter ExtensionFilter NameFilter AndFilter
                                                         └── filters: List<Filter>
```

The flat `directories` registry on `FileSearchAPI` mirrors the flat `showtimes` map in a booking system. Without it, locating `/home/user/docs` by path string would mean walking the tree from root on every search call. With it, every lookup is O(1).

---

## Class Design

Top-down: `FileSearchAPI` first, then `Directory` and `File`, then the `Filter` interface and concrete implementations, then `AndFilter`.

### FileSearchAPI

**Deriving State**

| Requirement | What FileSearchAPI must track |
|-------------|-------------------------------|
| "Specify a root directory path" | Directory objects indexed by path for O(1) lookup |

**Deriving Behavior**

| Need from requirements | Method on FileSearchAPI |
|------------------------|------------------------|
| "Search from a root path with filters" | `search(rootPath, filters) -> List<File>` |
| "Build the in-memory file system" | `addDirectory(parentPath, name) -> Directory` |
| "Build the in-memory file system" | `addFile(directoryPath, file) -> void` |

```
class FileSearchAPI:
    - directories: Map<String, Directory>

    + FileSearchAPI()
    + addDirectory(parentPath, name) -> Directory
    + addFile(directoryPath, file) -> void
    + search(rootPath, filters: List<Filter>) -> List<File>
    - traverse(directory, filters, results) -> void      // internal recursive DFS
```

---

### Directory

**Deriving State**

| Requirement | What Directory must track |
|-------------|--------------------------|
| "Recursive search through subdirectories" | List of child Directories |
| "Files only in results" | List of child Files |
| "O(1) lookup by path in FileSearchAPI" | Its own path string |

```
class Directory:
    - name:           String
    - path:           String
    - files:          List<File>
    - subdirectories: List<Directory>

    + Directory(name, path)
    + getName()            -> String
    + getPath()            -> String
    + getFiles()           -> List<File>
    + getSubdirectories()  -> List<Directory>
    + addFile(file)        -> void
    + addSubdirectory(dir) -> void
```

---

### File

**Deriving State**

| Requirement | What File must track |
|-------------|----------------------|
| "Name matching" | name |
| "Extension/format matching" | extension (stored without leading dot) |
| "Size filtering" | size in bytes |
| "Return file path in results" | full path including filename |

```
class File:
    - name:      String
    - extension: String    // stored without leading dot, e.g. "xml" not ".xml"
    - size:      long      // bytes
    - path:      String    // full path including filename

    + File(name, extension, size, path)
    + getName()      -> String
    + getExtension() -> String
    + getSize()      -> long
    + getPath()      -> String
```

Extension is stored without the leading dot so `ExtensionFilter` can compare `"xml"` against `"xml"` rather than `".xml"` against `"xml"`. The filter normalizes its input at construction — `File` never needs to know.

---

### Filter (interface)

The single-method interface that every filter implements. The entire extensibility story is here — adding a new filter type means implementing this interface and nothing else. `FileSearchAPI` never needs to change.

```
interface Filter:
    + apply(file: File) -> boolean
```

---

### MinSizeFilter

```
class MinSizeFilter implements Filter:
    - minBytes: long

    + MinSizeFilter(minBytes: long)
    + apply(file) -> boolean    // file.getSize() > minBytes
```

---

### MaxSizeFilter

```
class MaxSizeFilter implements Filter:
    - maxBytes: long

    + MaxSizeFilter(maxBytes: long)
    + apply(file) -> boolean    // file.getSize() < maxBytes
```

---

### ExtensionFilter

```
class ExtensionFilter implements Filter:
    - extension: String    // stored lowercase, without leading dot

    + ExtensionFilter(extension: String)    // strips "." prefix, lowercases at construction
    + apply(file) -> boolean                // file.getExtension().equalsIgnoreCase(this.extension)
```

Normalization at construction (stripping any leading dot, lowercasing) means `apply()` is a simple equality check with no repeated string manipulation per file.

---

### NameFilter

```
class NameFilter implements Filter:
    - nameContains: String    // stored lowercase

    + NameFilter(nameContains: String)    // lowercases at construction
    + apply(file) -> boolean              // file.getName().toLowerCase().contains(this.nameContains)
```

---

### AndFilter

```
class AndFilter implements Filter:
    - filters: List<Filter>

    + AndFilter(filters: List<Filter>)
    + apply(file) -> boolean    // returns true only if every child filter returns true
```

`AndFilter` is strictly a convenience composite. `FileSearchAPI.search()` already applies a `List<Filter>` with AND semantics internally. `AndFilter` is useful when the caller wants to encapsulate a reusable composed filter as a single object and hand it around — or once `OrFilter` exists and you need to nest AND-inside-OR in a filter tree.

---

### Final Class Design

```
+---------------------------------------+
|           FileSearchAPI               |
+---------------------------------------+
| - directories: Map<String, Directory> |
+---------------------------------------+
| + addDirectory(parentPath, name)      |
| + addFile(directoryPath, file)        |
| + search(rootPath, filters)           |
|       -> List<File>                   |
| - traverse(dir, filters, results)     |
+---------------------------------------+
              |
        owns  |
              v
+----------------------+        +---------------------+
|      Directory       |        |        File         |
+----------------------+        +---------------------+
| - name: String       | owns   | - name: String      |
| - path: String       |------->| - extension: String |
| - files: List<File>  |        | - size: long        |
| - subdirectories:    |        | - path: String      |
|     List<Directory>  |        +---------------------+
+----------------------+
| + addFile()          |
| + addSubdirectory()  |
+----------------------+

              <<interface>>
                 Filter
         + apply(file) -> boolean
         ^        ^         ^         ^         ^
         |        |        |          |         |
   MinSize   MaxSize   Extension  NameFilter  AndFilter
   Filter    Filter    Filter                 └── filters: List<Filter>
  -minBytes -maxBytes  -extension  -nameContains   (applies all, short-circuits)
```

---

## Implementation

The three most interesting methods are `FileSearchAPI.search()` (entry point and validation), the private `traverse()` (recursive DFS core), and `AndFilter.apply()` (composite delegation). We'll also show a concrete filter to make the pattern concrete.

### search()

**Core logic:**
1. Look up the root directory by path in the flat registry
2. Validate it exists — throw early rather than silently returning empty results
3. Kick off recursive traversal with an accumulator list
4. Return the accumulated results

**Edge cases:**
- Root path not found in the registry — throw
- Empty filters list — treated as "match everything"; no filter rejects any file

```
search(rootPath, filters)
    directory = directories[rootPath]
    if directory == null
        throw Error("Directory not found: " + rootPath)

    results = []
    traverse(directory, filters, results)
    return results
```

---

### traverse()

The recursive DFS. For each file in the current directory, checks all filters and adds matches to results. Then recurses into each subdirectory.

**Core logic:**
1. For each file in this directory: if all filters pass, add to results
2. Recurse into each subdirectory

**Edge cases:**
- Directory has no files — skip the file loop, still recurse into subdirectories
- Directory has no subdirectories — recursion terminates naturally
- Empty filters list — `matchesAll` returns true for every file

```
traverse(directory, filters, results)
    for file in directory.getFiles()
        if matchesAll(file, filters)
            results.add(file)

    for subdir in directory.getSubdirectories()
        traverse(subdir, filters, results)

matchesAll(file, filters)
    for filter in filters
        if !filter.apply(file)
            return false      // short-circuit on first failure
    return true
```

`matchesAll` short-circuits on the first failing filter. If a file fails `ExtensionFilter` immediately, we never evaluate `MinSizeFilter`. Per-file cost is proportional to how early a filter rejects, not total filter count.

---

### ExtensionFilter — construction and apply

```
ExtensionFilter(extension)
    normalized = extension.startsWith(".")
        ? extension.substring(1)
        : extension
    this.extension = normalized.toLowerCase()

apply(file)
    return file.getExtension().toLowerCase() == this.extension
```

---

### AndFilter.apply()

```
AndFilter(filters)
    this.filters = filters

apply(file)
    for filter in this.filters
        if !filter.apply(file)
            return false
    return true
```

Same short-circuit logic as `matchesAll` — stops at first failure. A child `AndFilter` inside an `OrFilter` applies identically, which is what makes the composite tree work without any special casing.

---

## Verification

**File system setup:**
```
/root
  /docs
    report.xml    (size: 8MB,  extension: xml)
    notes.txt     (size: 2MB,  extension: txt)
    data.xml      (size: 3MB,  extension: xml)
  /images
    photo.png     (size: 10MB, extension: png)
    logo.xml      (size: 1MB,  extension: xml)
```

**Search 1: All XML files greater than 5MB**
```
search("/root", [ExtensionFilter("xml"), MinSizeFilter(5MB)])

traverse("/root", ...)
  /root has no direct files — skip file loop
  recurse into /docs:
    report.xml:  ExtensionFilter → "xml" == "xml"  ✓
                 MinSizeFilter   → 8MB > 5MB        ✓  → add
    notes.txt:   ExtensionFilter → "txt" == "xml"   ✗  → skip (short-circuit)
    data.xml:    ExtensionFilter → "xml" == "xml"   ✓
                 MinSizeFilter   → 3MB > 5MB         ✗  → skip
  recurse into /images:
    photo.png:   ExtensionFilter → "png" == "xml"   ✗  → skip
    logo.xml:    ExtensionFilter → "xml" == "xml"   ✓
                 MinSizeFilter   → 1MB > 5MB         ✗  → skip

Result: [report.xml]  ✓
```

**Search 2: All XML files — caller passes ".xml" with leading dot**
```
search("/root", [ExtensionFilter(".xml")])

ExtensionFilter(".xml") → strips dot → stores "xml" at construction

traverse: checks extension "xml" == "xml" for report.xml, data.xml, logo.xml

Result: [report.xml, data.xml, logo.xml]  ✓
```

**Search 3: Files smaller than 5MB**
```
search("/root", [MaxSizeFilter(5MB)])

report.xml:  8MB < 5MB   ✗  → skip
notes.txt:   2MB < 5MB   ✓  → add
data.xml:    3MB < 5MB   ✓  → add
photo.png:  10MB < 5MB   ✗  → skip
logo.xml:    1MB < 5MB   ✓  → add

Result: [notes.txt, data.xml, logo.xml]  ✓
```

**Search 4: No filters — returns all files**
```
search("/root", [])

matchesAll(file, []) → empty loop → returns true for every file

Result: [report.xml, notes.txt, data.xml, photo.png, logo.xml]  ✓
```

**Search 5: Directory not found**
```
search("/nonexistent", [ExtensionFilter("xml")])
  directories["/nonexistent"] == null
  → throw Error("Directory not found: /nonexistent")  ✓
```

---

## Extensibility

### 1. "How would you add a new filter — say, last modified date?"

> "I'd implement `Filter` with a `ModifiedAfterFilter` class. It stores a `cutoffTime: DateTime` at construction and `apply(file)` returns `file.getLastModified() > cutoffTime`. `File` would need a `lastModified` field added. That's it — `FileSearchAPI`, `traverse()`, `AndFilter`, and every existing filter are completely untouched. This is the Strategy pattern working as intended: adding a filter is purely additive, and the caller just includes it in the list passed to `search()`."

```
class ModifiedAfterFilter implements Filter:
    - cutoffTime: DateTime

    + ModifiedAfterFilter(cutoffTime)
    + apply(file) -> boolean
        return file.getLastModified() > cutoffTime
```

---

### 2. "How would you support OR logic — 'XML files OR files greater than 5MB'?"

Right now multiple filters always compose with AND. OR requires a different combinator.

> "I'd add an `OrFilter` composite that mirrors `AndFilter` but stops at the first passing filter instead of the first failing one. The `Filter` interface is unchanged, so every existing filter plugs directly into `OrFilter` without modification. For nested AND-inside-OR — like 'files that are (XML AND > 5MB) OR (PNG AND > 10MB)' — wrap two `AndFilter` instances inside one `OrFilter`. The tree composes cleanly because all nodes are just `Filter` implementations; no class needs to know whether its parent is an AND or OR node."

```
class OrFilter implements Filter:
    - filters: List<Filter>

    + OrFilter(filters: List<Filter>)
    + apply(file) -> boolean
        for filter in filters
            if filter.apply(file)
                return true     // short-circuit on first match
        return false
```

---

### 3. "How would you add a maximum search depth?"

Right now `traverse()` recurses fully. A `maxDepth` of 0 should return only files directly in the root directory; depth 1 includes one level of subdirectories, and so on.

> "I'd add an optional `maxDepth` parameter to `search()`, defaulting to -1 for unlimited. I'd thread a `currentDepth` counter into `traverse()` and skip the subdirectory loop when the depth limit is reached. No filter class is affected — this is purely a traversal concern. The only changes are in `FileSearchAPI.search()` and the private `traverse()` signature."

```
search(rootPath, filters, maxDepth = -1)
    directory = directories[rootPath]
    if directory == null
        throw Error("Directory not found: " + rootPath)
    results = []
    traverse(directory, filters, results, 0, maxDepth)
    return results

traverse(directory, filters, results, currentDepth, maxDepth)
    for file in directory.getFiles()
        if matchesAll(file, filters)
            results.add(file)

    if maxDepth != -1 && currentDepth >= maxDepth
        return                  // stop recursing at depth limit

    for subdir in directory.getSubdirectories()
        traverse(subdir, filters, results, currentDepth + 1, maxDepth)
```
