# Skill: `existing_package_loader`

**Phase:** Planning Agent
**Mode:** Read-only

---

## Purpose

Loads the current package partition from annotated copy-map files. This is the weighted prior for all package assignment proposals. The loader distinguishes between locked packages (immutable) and unlocked packages (open to reassignment proposals).

This skill MUST run before any package assignment logic. The agent must not propose assignments without first loading the existing partition.

---

## Inputs

| Input | Type | Required | Description |
|---|---|---|---|
| `copyMapDir` | string | Yes | Directory containing annotated copy-map files |
| `lockFile` | string | No | Path to a JSON file listing locked package names; default: `{copyMapDir}/locked-packages.json` |
| `viewPath` | string | Yes | ClearCase view root for resolving relative paths |

---

## Outputs

Produces an in-memory `ExistingPackageMap` object:

```typescript
interface ExistingPackageMap {
  packages: ExistingPackage[];
  unassignedFiles: string[];       // Files in view with no copy-map entry
  totalFilesLoaded: number;
  lockedPackageCount: number;
}

interface ExistingPackage {
  packageName: string;
  locked: boolean;
  files: ExistingFileAssignment[];
  copyMapSource: string;           // Which copy-map file this came from
}

interface ExistingFileAssignment {
  filePath: string;    // Relative to view root
  annotation?: string; // Optional human annotation from copy-map (e.g., "keep here", "review")
}
```

---

## Behavior

### 1. Discover Copy-Map Files

Enumerate all files in `copyMapDir` matching the copy-map file format.

> [OPEN] §3.1 — The copy-map file format is unknown. The loader cannot be implemented until the format is confirmed. See `open-questions.md §3.1`.

### 2. Parse Each Copy-Map File

For each copy-map file, extract:
- The package name this file defines
- The list of source files assigned to the package
- Any annotations on individual file assignments

### 3. Load Locked Package List

Read `lockFile` (JSON array of package names) to determine which packages are locked:

```json
["libplatform-stable", "libcore-v1", "liblegacy"]
```

If `lockFile` does not exist, assume no packages are locked and emit an `info` warning.

### 4. Build the Package Map

Combine parsed copy-map entries with lock status to produce `ExistingPackageMap`.

### 5. Detect Conflicts in Existing Partition

If the same file path appears in two different copy-map files (and neither package is marked as forkable for that file), flag as a conflict. Report to the user but do not halt — the Planning Agent will surface this as a `conflict` assignment in `PackageAssignment.json`.

### 6. Identify Unassigned Files

Compare the set of files in the view (from `repo_snapshot`) against the set of files with copy-map entries. Files with no copy-map entry are listed as `unassignedFiles`.

> **Note:** Orphan file handling (files with no copy-map AND no build record) is governed by `open-questions.md §5.2`.

### 7. Log Invocation

```json
{
  "skill": "existing_package_loader",
  "phase": "complete",
  "outputs": {
    "packageCount": 47,
    "lockedPackageCount": 8,
    "totalFilesLoaded": 2341,
    "unassignedFileCount": 156
  }
}
```

---

## Error Handling

| Error Condition | Behavior |
|---|---|
| `copyMapDir` does not exist | Halt with error. Cannot run planning without an existing partition. |
| Copy-map format unrecognized | Halt with error. See `open-questions.md §3.1`. |
| `lockFile` missing | Emit `info` warning: "No lock file found; treating all packages as unlocked." Continue. |
| Duplicate file in two copy-maps | Emit `warning`. Record as conflict. Continue. |
| Zero packages loaded | Halt with error: "No copy-map files found. Is the copy-map directory correct?" |

---

## Locked Package Guarantee

The `ExistingPackageMap.packages[*].locked` field is the source of truth for the lock status throughout the entire agent run. All downstream skills MUST check this field before proposing any change to a package's file membership.

**Enforcement:**
- Assignment logic: never propose a file move to/from a locked package
- `copy_map_generator`: never emit a copy-map for a locked package
- `cmake_generator`, `conan_generator`: never generate files for a locked package

---

## Worked Example

**Inputs:**
- `copyMapDir`: `/repo/copy-maps/`
- `lockFile`: `/repo/copy-maps/locked-packages.json`

**Files in `copyMapDir`:**
```
/repo/copy-maps/libcore.copymap
/repo/copy-maps/libplatform.copymap
/repo/copy-maps/locked-packages.json
```

**`locked-packages.json`:**
```json
["libplatform"]
```

**`libcore.copymap`** (format TBD — see `open-questions.md §3.1`):
```
src/core/allocator.cpp
src/core/memory.cpp
include/core/allocator.h   # keep here
```

**`libplatform.copymap`**:
```
src/platform/thread.cpp
include/platform/thread.h
```

**Output (in-memory `ExistingPackageMap`):**
```json
{
  "packages": [
    {
      "packageName": "libcore",
      "locked": false,
      "files": [
        { "filePath": "src/core/allocator.cpp" },
        { "filePath": "src/core/memory.cpp" },
        { "filePath": "include/core/allocator.h", "annotation": "keep here" }
      ],
      "copyMapSource": "libcore.copymap"
    },
    {
      "packageName": "libplatform",
      "locked": true,
      "files": [
        { "filePath": "src/platform/thread.cpp" },
        { "filePath": "include/platform/thread.h" }
      ],
      "copyMapSource": "libplatform.copymap"
    }
  ],
  "unassignedFiles": ["src/util/hash.cpp"],
  "totalFilesLoaded": 5,
  "lockedPackageCount": 1
}
```
