# Skill: `dependency_breaker`

**Phase:** Implementation Agent
**Mode:** Read-write (proposes patches; never auto-applies)

---

## Purpose

Proposes narrow, mechanical code edits to break circular package dependencies identified by the Planning Agent. This skill does NOT perform general refactoring. It targets only the specific include relationships that form cycles, and only uses low-risk, reviewable strategies.

All proposals are emitted as `PatchProposal.json` entries. Patches are NEVER applied automatically. A human must review and approve each patch before it is applied.

---

## Inputs

| Input | Type | Required | Description |
|---|---|---|---|
| `dependencyGraph` | `DependencyGraph` | Yes | Output of `fw_include_graph`, containing `sccs[]` |
| `sccId` | string | Yes | Which SCC (cycle) to target |
| `packageAssignment` | `PackageAssignment` | Yes | Accepted package assignments |
| `viewPath` | string | Yes | ClearCase view root for reading source file contents |
| `conservatism` | string | No | `"low"` (default) or `"medium"` — see `open-questions.md §5.5` |

---

## Outputs

Appends to `PatchProposal.json`:

```typescript
interface PatchProposal {
  proposals: Patch[];
}

interface Patch {
  id: string;
  patchType: "dependency_break" | "conan_patch";
  targetFile: string;
  rationale: string;
  unifiedDiff: string;
  riskLevel: "low" | "medium" | "high";
  approvalStatus: "pending" | "approved" | "rejected";
  sccRef: string;          // Which SCC this patch addresses
  strategy: string;        // e.g., "extract_interface_header"
}
```

---

## Behavior

### 1. Load the Target SCC

Read the `sccs[]` entry for `sccId` from `DependencyGraph.json`. Identify:
- The cycle members (packages)
- The minimum cut edges (the specific include edges whose removal breaks the cycle)
- The proposed resolutions (ranked by `fw_include_graph`)

### 2. Evaluate Applicable Strategies

For `conservatism: "low"`, only attempt `strategy: "extract_interface_header"`:

**Extract Interface Header Strategy:**

Given an include edge `(includer_file, includee_header)` that crosses from package P2 to package P1 (where this edge completes the cycle):

1. Identify what the `includer_file` actually uses from `includee_header` (types, forward declarations, constants)
2. If the usage is limited to: forward declarations, enums, `typedef`s, or `#define` constants — the edit is low-risk
3. Propose: create a new header `includee_header_fwd.h` containing only those declarations
4. Propose: replace `#include "includee_header.h"` in `includer_file` with `#include "includee_header_fwd.h"`
5. Move `includee_header_fwd.h` to a package that neither P1 nor P2 depends on (or a new shared package)

For `conservatism: "medium"`, additionally attempt `strategy: "introduce_indirection"` (introduce an indirection header in a shared package).

**Never attempt:**
- Merging packages (too high impact — requires re-running planning)
- Creating new shared packages (too high impact — requires human architectural decision)
- Splitting a header into multiple files beyond extracting forward declarations
- Any edit that modifies function signatures, class definitions, or member layouts

### 3. Validate Proposed Edits

For each proposed edit:
- The new file must not already exist
- The unified diff must be valid and apply cleanly to the current file
- The edit must not remove any symbol that `includer_file` currently uses (validate using include graph)

### 4. Assign Risk Level

- `low`: only forward declarations extracted, no logic changed
- `medium`: typedef or enum extracted (may affect ABI)
- `high`: any other change — if reached, emit a `high`-risk warning and require explicit confirmation before including in proposal

If `conservatism: "low"` and risk level would be `medium` or `high`, skip the proposal and emit a `warning` to the user explaining why.

### 5. Emit Patch Proposal

For each valid proposed edit, add an entry to `PatchProposal.json` with `approvalStatus: "pending"`.

Include in the PR description a summary of each proposal with:
- The cycle being broken
- The files affected
- The strategy used
- The risk level
- A reminder that no patch is applied until explicitly approved

---

## Error Handling

| Error Condition | Behavior |
|---|---|
| SCC has no minimum cut edges (fully connected) | Report: "Cycle cannot be broken by a single edge removal. Consider merging packages." Emit no proposal. |
| Usage analysis cannot determine what is used from includee | Emit `high`-risk proposal with note: "Unable to verify usage scope — human review required" |
| Proposed new header already exists | Skip and report conflict |
| Edit does not apply cleanly | Skip and report |

---

## Worked Example

**Scenario:** `libcore` and `libutil` are in a cycle (from `fw_include_graph` worked example).

**Minimum cut edge:** `src/util/buffer.cpp` → `include/core/types.h`

**Analysis of `include/core/types.h`:**
- `src/util/buffer.cpp` uses: `struct Size` (a simple struct with two `int` fields) and `typedef unsigned int uint32_t`

**Strategy: extract interface header**

1. Create `include/core/types-fwd.h` with:
```c
#ifndef CORE_TYPES_FWD_H
#define CORE_TYPES_FWD_H
typedef unsigned int uint32_t;
struct Size { int width; int height; };
#endif
```

2. Modify `src/util/buffer.cpp`: replace `#include "include/core/types.h"` with `#include "include/core/types-fwd.h"`

3. Add `include/core/types-fwd.h` to a new `libcore-types` package (or `libshared`) — requires user decision on which package to use.

**Patch proposal emitted:**
```json
{
  "id": "patch-scc-0-extract-types-fwd",
  "patchType": "dependency_break",
  "targetFile": "src/util/buffer.cpp",
  "rationale": "Breaks cycle scc-0 (libcore ↔ libutil) by replacing full include with forward-declarations-only header.",
  "riskLevel": "low",
  "approvalStatus": "pending",
  "sccRef": "scc-0",
  "strategy": "extract_interface_header",
  "unifiedDiff": "--- a/src/util/buffer.cpp\n+++ b/src/util/buffer.cpp\n@@ -1,4 +1,4 @@\n-#include \"include/core/types.h\"\n+#include \"include/core/types-fwd.h\"\n ..."
}
```
