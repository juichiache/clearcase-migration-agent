# Skill: `header_pruner`

**Phase:** Planning Agent
**Mode:** Read-only

---

## Purpose

Identifies unused `#include` directives in source files and flags headers that are missing from a file's own includes but are currently only reachable transitively (via another included header). Both types of findings are reported for human review — the skill does not modify source files.

---

## Inputs

| Input | Type | Required | Description |
|---|---|---|---|
| `includeGraph` | `IncludeGraph` | Yes | Output of `fw_include_graph` |
| `buildRecordSet` | `BuildRecordSet` | Yes | Output of `build_record_ingester` |
| `packageAssignment` | Draft `PackageAssignment` | Yes | Current package-to-file mapping |

---

## Outputs

Produces an in-memory `HeaderPrunerReport` (included in planning summary; not persisted as a standalone artifact):

```typescript
interface HeaderPrunerReport {
  unusedIncludes: UnusedIncludeFinding[];
  transitiveIncludes: TransitiveIncludeFinding[];
  summary: {
    totalFilesAnalyzed: number;
    filesWithUnusedIncludes: number;
    filesWithTransitiveIncludes: number;
  };
}

interface UnusedIncludeFinding {
  filePath: string;
  package: string;
  unusedHeaders: string[];        // Headers included but not used
  macroContext: Record<string, string>; // Which macro combo reveals this
  confidence: number;              // 0.0–1.0; lower if macro analysis is incomplete
  note?: string;
}

interface TransitiveIncludeFinding {
  filePath: string;
  package: string;
  symbol: string;                 // Symbol used from the transitive header (if known)
  transitiveHeader: string;       // The header providing the symbol
  directProvider?: string;        // The directly included header that re-exports it
  macroContext: Record<string, string>;
  severity: "warning" | "info";   // "warning" if the transitive path could break on restructuring
}
```

---

## Behavior

### 1. Build the Reachability Map

For each source file S and each macro combination C:
  - `directIncludes(S, C)` = set of headers directly included by S under C (from build records)
  - `reachableIncludes(S, C)` = transitive closure of directIncludes (recursively expand each header's includes)

### 2. Identify Unused Includes

A header H is "unused" in file S under macro combo C if:
- H ∈ `directIncludes(S, C)`
- No symbol from H is referenced in S (symbol reference analysis)

**Note:** Full symbol reference analysis requires parsing the source file. For the v1 agent, unused include detection is approximated using the include graph alone:

**Approximation rule (v1):** If H appears in `directIncludes(S, C)` but H's own includes contribute zero edges to other headers that S uses, H is *likely* unused. This will have false positives (H may define macros or types used inline). Flag with `confidence ≤ 0.7` and `note: "Approximate — full symbol analysis not performed"`.

### 3. Identify Transitive Includes

A header T is a "transitive include" for file S under macro combo C if:
- T ∉ `directIncludes(S, C)` (S does not directly include T)
- T ∈ `reachableIncludes(S, C)` (T is reachable via other headers)
- S uses symbols from T (or if symbol analysis is unavailable: T is used by other packages in S's package)

Flag these with `severity: "warning"` if the transitive path crosses package boundaries (i.e., S is in package P1, T is in package P2, and the intermediate header is in package P3 — restructuring P3 could break S).

### 4. Cross-Package Transitive Risk

The most important finding type: a transitive include that crosses package boundaries. If file S in `libcore` reaches header T in `libplatform` only via a header in `libutil`, then:
- If `libutil` is restructured, S may break
- Flag as `severity: "warning"` with description: "S depends on T via libutil — this dependency will break if libutil's includes are reorganized"

### 5. Write to Planning Summary

Append findings to the plan PR summary as a section: "Header Findings". Do not block planning on these findings — they are informational for the team.

---

## Error Handling

| Error Condition | Behavior |
|---|---|
| Include graph incomplete (unresolved headers) | Analyze with available data; note in findings that some headers were not resolved |
| No build records for a file | Skip that file; emit `info` in summary |
| Symbol analysis not available | Use approximation; emit `confidence ≤ 0.7` |

---

## Worked Example

**Scenario:**

File `src/core/allocator.cpp` in package `libcore` directly includes:
- `include/core/allocator.h` (used — defines `Allocator` class)
- `include/util/logging.h` (used — `LOG_DEBUG` macro)
- `include/platform/compat.h` (NOT used directly — only included because another dev "just in case")

`include/util/logging.h` itself includes `include/platform/debug.h`, which defines `DEBUG_LEVEL`.
`src/core/allocator.cpp` uses `DEBUG_LEVEL` but does not directly include `include/platform/debug.h`.

**Findings:**

1. **Unused include:**
   ```json
   {
     "filePath": "src/core/allocator.cpp",
     "package": "libcore",
     "unusedHeaders": ["include/platform/compat.h"],
     "confidence": 0.65,
     "note": "Approximate — full symbol analysis not performed"
   }
   ```

2. **Transitive include:**
   ```json
   {
     "filePath": "src/core/allocator.cpp",
     "package": "libcore",
     "symbol": "DEBUG_LEVEL",
     "transitiveHeader": "include/platform/debug.h",
     "directProvider": "include/util/logging.h",
     "severity": "warning",
     "note": "libcore reaches libplatform/debug.h only via libutil/logging.h. If libutil is restructured, this dependency may break."
   }
   ```
