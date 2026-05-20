# Skill: `pr_reviewer`

**Phase:** Implementation Agent
**Mode:** Read-only

---

## Purpose

Reviews an open package PR for two classes of issues:

1. **Include audit:** Unused includes in generated/modified files; missing includes that are only reachable transitively.
2. **Duplicate assignment audit:** Files that are assigned to multiple packages (unless explicitly marked forkable).

This skill posts its findings as inline PR review comments. It does not modify code.

---

## Inputs

| Input | Type | Required | Description |
|---|---|---|---|
| `prNumber` | number | Yes | The PR to review |
| `packageName` | string | Yes | The package this PR implements |
| `packageAssignment` | `PackageAssignment` | Yes | Accepted assignments for context |
| `includeGraph` | `IncludeGraph` | Yes | From Planning Agent's run |
| `forkableFiles` | string[] | No | File paths allowed in multiple packages (see `open-questions.md §5.1`) |

---

## Outputs

- Inline PR review comments on the `prNumber` PR
- A top-level PR review comment with a summary
- An entry in `RunLogEvent.jsonl`

This skill does NOT produce a standalone artifact file (findings are posted directly to the PR).

---

## Behavior

### 1. Fetch PR Diff

Read the diff of `prNumber` via the GitHub API. Extract:
- Modified files and their line-by-line changes
- New files added (CMakeLists.txt, conanfile.py, copy-map entries)

### 2. Include Audit

For each source file in the PR's scope (all files in `packageAssignment` for `packageName`):

**Check for unused includes:**
- Cross-reference with `HeaderPrunerReport` output from the Planning Agent (if available)
- For any include flagged as unused with `confidence ≥ 0.7`, post an inline comment on the relevant line:
  > ⚠️ **Possible unused include:** `{header}` appears unused in this file under macro combo `{macros}`. Consider removing if not needed for side effects.

**Check for missing includes:**
- Cross-reference with `TransitiveIncludeFinding` entries from `HeaderPrunerReport`
- For any file where a transitive include crosses package boundaries (severity: `"warning"`), post a comment:
  > ⚠️ **Transitive include dependency:** `{filePath}` uses symbols from `{transitiveHeader}` (in package `{package}`) only via `{directProvider}`. Add a direct `#include` for robustness, or accept the transitive dependency.

### 3. Duplicate Assignment Audit

For each file listed in the PR's copy-map entries:
  - Check if the file also appears in any OTHER package's accepted `PackageAssignment` entries
  - If yes, and the file is NOT in `forkableFiles`:
    - Post a top-level PR comment (not inline — the conflict is in the copy-map, not the source file):
      > ❌ **Duplicate assignment conflict:** `{filePath}` is assigned to both `{packageName}` and `{otherPackage}`. This must be resolved before merging. See `PackageAssignment.json` entry for details.
  - If yes, and the file IS in `forkableFiles`: no comment needed.

### 4. Summarize

Post a top-level review comment summarizing all findings:

```
## PR Review Summary — {packageName}

**Include Audit:**
- {N} possible unused includes (confidence ≥ 70%)
- {M} transitive include warnings (cross-package boundary)

**Duplicate Assignment Audit:**
- {K} duplicate assignment conflicts ❌

{If K > 0}: ❌ This PR cannot be merged until all duplicate conflicts are resolved.
{If K == 0 and N == 0 and M == 0}: ✅ No issues found.
{If K == 0 and (N > 0 or M > 0)}: ⚠️ {N+M} warnings to review. Merge is not blocked but review is recommended.
```

### 5. Set Review Status

- If any `K > 0` (duplicate conflict): submit review with `REQUEST_CHANGES`
- If no duplicates but warnings exist: submit review with `COMMENT`
- If no issues: submit review with `APPROVE`

---

## Error Handling

| Error Condition | Behavior |
|---|---|
| PR not found | Halt: "PR #{prNumber} not found. Check the PR number and repository." |
| `HeaderPrunerReport` not available | Skip include audit; note in summary: "Include audit skipped — header pruner report not available for this run." |
| GitHub API rate limit | Wait with exponential backoff (max 3 retries), then halt |
| PR already has a review from this agent | Update the existing review comment instead of creating a new one |

---

## Worked Example

**PR #47 — package `libcore`**

**Copy-map entries in PR:** `src/core/allocator.cpp`, `src/core/memory.cpp`, `include/core/allocator.h`

**Planning Agent's `HeaderPrunerReport`:**
- `src/core/allocator.cpp` has unused include `include/platform/compat.h` (confidence 0.72)
- `src/core/allocator.cpp` has transitive include: uses `DEBUG_LEVEL` from `include/platform/debug.h` via `include/util/logging.h`

**`PackageAssignment.json`:**
- `include/core/allocator.h` is also in `libcore-legacy` (a separate package)

**Posted PR comments:**

Inline on `src/core/allocator.cpp` line 5 (`#include "include/platform/compat.h"`):
> ⚠️ **Possible unused include:** `include/platform/compat.h` appears unused in this file. Consider removing if not needed for side effects. (Confidence: 72%)

Inline on `src/core/allocator.cpp` line 12 (macro combo with `include/util/logging.h`):
> ⚠️ **Transitive include dependency:** `src/core/allocator.cpp` uses `DEBUG_LEVEL` from `include/platform/debug.h` (in package `libplatform`) only via `include/util/logging.h`. Add a direct `#include <platform/debug.h>` for robustness.

Top-level conflict comment:
> ❌ **Duplicate assignment conflict:** `include/core/allocator.h` is assigned to both `libcore` and `libcore-legacy`. Resolve this in `PackageAssignment.json` before merging.

**Summary comment:**
```
## PR Review Summary — libcore

**Include Audit:**
- 1 possible unused include (confidence ≥ 70%)
- 1 transitive include warning (cross-package boundary)

**Duplicate Assignment Audit:**
- 1 duplicate assignment conflict ❌

❌ This PR cannot be merged until all duplicate conflicts are resolved.
```

**Review submitted with:** `REQUEST_CHANGES`
