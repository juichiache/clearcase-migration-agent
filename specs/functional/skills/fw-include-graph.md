# Skill: `fw_include_graph`

**Phase:** Planning Agent
**Mode:** Read-only

---

## Purpose

Builds a directed include graph for the codebase with **macro context on every edge**. Each edge is a triple `(includer, includee, macro_set)` capturing the macro combination under which the include is active. This graph is the foundation for package dependency analysis and circular dependency detection.

---

## Inputs

| Input | Type | Required | Description |
|---|---|---|---|
| `buildRecordSet` | `BuildRecordSet` | Yes | Output of `build_record_ingester` |
| `existingPackageMap` | `ExistingPackageMap` | Yes | Output of `existing_package_loader` |
| `viewPath` | string | Yes | ClearCase view root for resolving header search paths |
| `includePaths` | string[] | No | Additional include search paths (in order); default: derived from view structure |

---

## Outputs

Produces `DependencyGraph.json` (written to run directory after schema validation):

```typescript
interface IncludeGraph {
  nodes: IncludeNode[];
  edges: IncludeEdge[];
  packageDependencies: PackageDependency[];
  sccs: StronglyConnectedComponent[];
}

interface IncludeNode {
  filePath: string;       // Relative to view root
  fileType: "source" | "header";
  package: string | null; // null if unassigned
  locked: boolean;
}

interface IncludeEdge {
  includer: string;       // File path
  includee: string;       // Resolved file path (not the literal #include string)
  macroSets: Record<string, string>[];  // Which macro combos activate this edge
  edgeStrength: number;   // 0.0–1.0; higher = more macro combos activate this edge
}

interface PackageDependency {
  fromPackage: string;
  toPackage: string;
  strength: number;       // Fraction of files in fromPackage that include toPackage files
  edges: string[];        // Include edge IDs that contribute to this dependency
  macroContext: Record<string, string>[];  // Union of macro sets across contributing edges
}

interface StronglyConnectedComponent {
  id: string;
  members: string[];      // Package names
  cycleEdges: CycleEdge[];
  proposedResolutions: Resolution[];
}

interface CycleEdge {
  fromPackage: string;
  toPackage: string;
  viaFiles: { includer: string; includee: string }[];
}

interface Resolution {
  rank: number;
  strategy: "extract_interface_header" | "merge_packages" | "introduce_indirection" | "new_shared_package";
  description: string;
  estimatedRisk: "low" | "medium" | "high";
  affectedFiles: string[];
}
```

---

## Behavior

### 1. Build File-Level Include Graph

For each source file in `BuildRecordSet`:
  - For each macro combination in the file's record:
    - For each `includedHeader` in that combo:
      - Resolve the header to an absolute path using `includePaths` (in order)
      - Create an `IncludeEdge` (or add the macro set to an existing edge for the same pair)

**Resolution order (standard C/C++ include search):**
1. Same directory as the including file
2. Include paths from `includePaths` (in order)
3. If still unresolved: emit `warning` and skip the edge

### 2. Resolve Ambiguous Headers

If the same `#include "foo.h"` resolves to different files under different macro sets:

> [OPEN] §5.3 — Resolution strategy for ambiguous headers. See `open-questions.md §5.3`.
> Default behavior until resolved: use resolution (a) — pick the more common resolution, flag as `conflict` in `PackageAssignment.json`.

### 3. Compute Edge Strength

For each file-level edge, `edgeStrength` = (number of macro combos where this edge is active) / (total macro combos for the including file).

An edge strength of 1.0 means the include is always active (macro-invariant). Strength < 1.0 means it's conditionally active.

### 4. Lift to Package-Level Graph

Group file-level edges by package assignment to produce `packageDependencies`. For each package pair (P1, P2) where at least one file in P1 includes at least one file in P2:
- `strength` = (files in P1 that include P2 files) / (total files in P1)
- `macroContext` = union of all macro sets across contributing edges

### 5. Detect SCCs (Tarjan's Algorithm)

Run Tarjan's SCC algorithm on the package-level graph. Each SCC with more than one member is a circular dependency.

For each SCC:
- List the members (package names)
- Identify the minimum cut edges (the smallest set of include edges whose removal breaks the cycle)
- Generate ranked resolution proposals (see §5.4 in `open-questions.md`)

### 6. Write Output

Validate against `DependencyGraph.schema.json`. Write to `artifacts/runs/{runId}/DependencyGraph.json`.

---

## Error Handling

| Error Condition | Behavior |
|---|---|
| Header cannot be resolved | Emit `warning`, skip edge, continue |
| All includes for a file unresolved | Emit `error` warning for that file; continue |
| Circular dependency detected | Add to `sccs[]`, do NOT halt — report to user via `user_dialog` |
| Schema validation failure | Halt, report schema path and violating value |

---

## Worked Example

**Scenario:** Two packages, `libcore` and `libutil`, with a circular dependency.

**Build records:**
- `src/core/allocator.cpp` → includes `include/util/buffer.h` (under all macro combos)
- `src/util/buffer.cpp` → includes `include/core/types.h` (under all macro combos)

**Package assignments:**
- `libcore`: `src/core/allocator.cpp`, `include/core/types.h`
- `libutil`: `src/util/buffer.cpp`, `include/util/buffer.h`

**Output (excerpt from `DependencyGraph.json`):**
```json
{
  "packageDependencies": [
    {
      "fromPackage": "libcore",
      "toPackage": "libutil",
      "strength": 1.0,
      "macroContext": [{}]
    },
    {
      "fromPackage": "libutil",
      "toPackage": "libcore",
      "strength": 1.0,
      "macroContext": [{}]
    }
  ],
  "sccs": [
    {
      "id": "scc-0",
      "members": ["libcore", "libutil"],
      "cycleEdges": [
        {
          "fromPackage": "libcore",
          "toPackage": "libutil",
          "viaFiles": [{ "includer": "src/core/allocator.cpp", "includee": "include/util/buffer.h" }]
        },
        {
          "fromPackage": "libutil",
          "toPackage": "libcore",
          "viaFiles": [{ "includer": "src/util/buffer.cpp", "includee": "include/core/types.h" }]
        }
      ],
      "proposedResolutions": [
        {
          "rank": 1,
          "strategy": "extract_interface_header",
          "description": "Extract forward declarations from include/core/types.h into a new include/core/types-fwd.h. Move types-fwd.h to a shared package. libutil depends on the shared package instead of libcore.",
          "estimatedRisk": "low",
          "affectedFiles": ["include/core/types.h", "src/util/buffer.cpp"]
        },
        {
          "rank": 2,
          "strategy": "merge_packages",
          "description": "Merge libcore and libutil into a single package libcore-util.",
          "estimatedRisk": "medium",
          "affectedFiles": []
        }
      ]
    }
  ]
}
```
