# Skill: `compiler_error_tracer`

**Phase:** Implementation Agent
**Mode:** Read-only (analysis) + produces `BuildErrorReport.json` and `PatchProposal.json`

---

## Purpose

Reads CI build failure logs from a package PR, classifies each error, diagnoses the root cause within the context of the package assignment and Conan options, and proposes the minimum fix. This skill is invoked after CI fails on a package PR.

---

## Inputs

| Input | Type | Required | Description |
|---|---|---|---|
| `buildLogUrl` | string | Yes | URL to the CI build log (or local path to log file) |
| `packageName` | string | Yes | The package whose CI failed |
| `packageAssignment` | `PackageAssignment` | Yes | Accepted assignments for context |
| `macroMatrix` | `MacroMatrix` | Yes | Current Conan options for the package |
| `prNumber` | number | Yes | The PR number to annotate with findings |

---

## Outputs

- `BuildErrorReport.json` appended to run directory
- `PatchProposal.json` appended for any proposed code patches
- PR comment on `prNumber` summarizing findings

---

## Behavior

### 1. Fetch and Parse Build Log

Read the build log. Split into individual compiler error messages. Each error message typically has the format:

```
{filePath}:{line}:{col}: error: {message}
```

Parse each error into:
- `filePath` — the source file with the error
- `line` — line number
- `errorText` — the raw error message

### 2. Classify Errors

For each parsed error, classify into one of:

| Error Type | Indicators |
|---|---|
| `missing_include` | `"No such file or directory"`, `"Cannot open source file"` |
| `wrong_package` | `"undefined reference to"` for a symbol that exists in another package's files |
| `undefined_symbol` | `"undefined reference to"` for a symbol not found anywhere in the package set |
| `missing_option` | `"is not defined"` or `"implicit declaration of function"` where the symbol is guarded by a macro not in `MacroMatrix.json` |
| `other` | Anything not matching above |

### 3. Diagnose Each Error

For each classified error:

**`missing_include`:**
  - Attempt to find the missing header in the package assignment and dependency graph
  - If found in package P2: proposed fix is "add P2 to the `requires` list in `conanfile.py` for this package"
  - If found nowhere: proposed fix is "check if the header is in a locked package or outside the migration scope"

**`wrong_package`:**
  - Find the symbol definition in `PackageAssignment.json`
  - If the defining file is in a different package than expected: proposed fix is "move {file} to {package}" (copy-map change)
  - If the symbol should be in this package but isn't compiled: proposed fix is "add {file} to {package} copy-map"

**`missing_option`:**
  - Find the macro guard around the symbol
  - Check if the macro is in `MacroMatrix.json` for this package
  - If not: proposed fix is "add macro combination {macros} to MacroMatrix.json and re-generate conanfile.py"
  - If yes: the option value is not being passed correctly — diagnose the Conan option mapping

**`undefined_symbol`:**
  - Search across all packages for the symbol definition
  - If found: this is likely a `wrong_package` misclassification — re-diagnose
  - If not found: may be a missing dependency not in the migration scope — flag for human

### 4. Write `BuildErrorReport.json`

Validate against `BuildErrorReport.schema.json`. Write to run directory.

### 5. Emit `PatchProposal.json` for Code Changes

If the proposed fix requires a source code change (not just a copy-map or option change), emit a `PatchProposal.json` entry with `patchType: "conan_patch"` or `"dependency_break"`.

**Never auto-apply patches.** Always require human approval via PR comment.

### 6. Post PR Comment

Post a structured comment to `prNumber` with:
- Summary: "{N} errors classified, {M} copy-map fixes proposed, {K} patches proposed"
- Table of errors with classification and proposed fix
- Links to `BuildErrorReport.json`

---

## Error Handling

| Error Condition | Behavior |
|---|---|
| Build log not accessible | Halt: "Cannot fetch build log from {url}. Check CI credentials." |
| Zero errors parsed from log | Emit warning: "Build log fetched but zero errors parsed. Log format may have changed." |
| Error classification confidence < 0.5 | Flag as `errorType: "other"`, describe in `diagnosis` field, request human review |
| Proposed fix introduces a new circular dependency | Flag with a warning: "Proposed fix may introduce a new dependency cycle — verify with Planning Agent" |

---

## Worked Example

**CI build log excerpt:**
```
src/core/allocator.cpp:42:10: error: 'include/util/buffer.h' file not found
src/core/allocator.cpp:108:5: error: use of undeclared identifier 'thread_init'; did you mean 'thread_create'?
```

**Classification:**
1. `missing_include` — `include/util/buffer.h` not found
2. `undefined_symbol` — `thread_init`

**Diagnosis:**
1. `include/util/buffer.h` is assigned to package `libutil` in `PackageAssignment.json`. `libcore`'s `conanfile.py` is missing `requires = ["libutil/1.0"]`.
   - Proposed fix: add `"libutil/1.0"` to `requires` in `libcore/conanfile.py`
   - Fix type: `option_change` (conanfile change, not source change)

2. `thread_init` is not found anywhere in the package set. `thread_create` is the correct function name in `libplatform`. Likely a naming mismatch from a recent API change.
   - Proposed fix: none (cannot auto-fix naming mismatches) — flag for human
   - Diagnosis: "Symbol `thread_init` not found in any package. Possible renamed API — check if `thread_create` in libplatform is the intended function."

**`BuildErrorReport.json` (excerpt):**
```json
{
  "packageName": "libcore",
  "prNumber": 47,
  "errors": [
    {
      "errorType": "missing_include",
      "location": { "file": "src/core/allocator.cpp", "line": 42 },
      "rawMessage": "'include/util/buffer.h' file not found",
      "diagnosis": "Header is in package libutil. libcore's conanfile.py is missing libutil as a requirement.",
      "proposedFix": {
        "type": "option_change",
        "description": "Add libutil/1.0 to requires in libcore/conanfile.py",
        "changeRef": "conanfile-requires-libutil"
      }
    },
    {
      "errorType": "undefined_symbol",
      "location": { "file": "src/core/allocator.cpp", "line": 108 },
      "rawMessage": "use of undeclared identifier 'thread_init'",
      "diagnosis": "Symbol thread_init not found in any package. thread_create in libplatform may be the intended function.",
      "proposedFix": null
    }
  ]
}
```
