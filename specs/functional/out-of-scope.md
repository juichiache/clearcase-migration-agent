# Out of Scope

This document explicitly lists what the migration agent will NOT do. These exclusions are firm. If a request falls into one of these categories, the agent should decline and explain that the capability is out of scope.

---

## Explicit Exclusions

### 1. ClearCase VOB → Git Migration

The agent does **not** migrate the ClearCase VOB to a GitHub monorepo. It does not:
- Import ClearCase file history into Git
- Replay VOB transactions as Git commits
- Convert ClearCase labels to Git tags
- Restructure the VOB directory tree into a Git-friendly layout

**In scope:** The agent reads files from a mounted ClearCase view. It writes generated build files to Git repositories (package repos + workspace repo). The source files themselves stay in ClearCase.

---

### 2. Full ClearCase History Replay

The agent does **not** replay the full ClearCase version history into Git. No `clearimport`, `clearfsimport`, or equivalent tool is invoked or wrapped.

---

### 3. General Source Code Refactoring

The agent does **not** perform general refactoring of C/C++ source files. It will not:
- Rename symbols
- Move functions between files
- Extract classes
- Apply modernization patterns (smart pointers, range-for, etc.)
- Inline constants or replace macros with `constexpr`

**In scope (narrow exception):** The `dependency_breaker` skill may propose narrow, mechanical edits specifically to break circular package dependencies (e.g., moving a forward declaration to a separate header). These are always subject to human approval and are never applied automatically. See `skills/dependency-breaker.md` for the exact scope.

---

### 4. Header Reorganization

The agent does **not** move header files between directories. It may:
- Identify unused includes (via `header_pruner`)
- Flag missing includes that are only reachable transitively
- Propose removing a `#include` from a source file

It will not:
- Move a header file to a new location
- Rename include paths
- Restructure include directories

> **Note:** Include path rewriting (if needed as part of the copy-map process) may be a separate step handled by the team's existing tooling. See `open-questions.md §3.6`.

---

### 5. `#define` Cleanup

The agent does **not** clean up preprocessor macros. It:
- Reads macro combinations from build records (read-only)
- Derives Conan options from observed combinations
- Does not propose to remove macros, unify definitions, or simplify conditional compilation

---

### 6. Conan Version Upgrades

The agent does **not** upgrade the Conan version in use. It generates `conanfile.py` for the Conan version specified by the team. Version selection is a human decision.

---

### 7. Running in GitHub Actions Cloud

The agent runs **on the engineer's machine** (via VS Code Remote-SSH). It is not designed to run as an unattended job in GitHub Actions cloud. Specifically, it does not:
- Mount ClearCase views in a cloud CI environment
- Operate without a human in the loop
- Auto-approve and auto-merge PRs without human review

The CI pipeline in GitHub Actions is for **validating** the output (Level 4 build test), not for running the agent itself.

---

### 8. Replacing omake

The agent does **not** replace the omake build system. The migration goal is CMake + Conan for the new package layout. omake may continue to be used during the transition period for products not yet migrated.

---

### 9. Build Record Extraction

The agent does **not** extract build records from the ClearCase source tree. The scripted YAML build records are produced by a separate Python extraction script owned by the build system team. The agent reads these records as inputs but does not generate them.

---

### 10. Automated Code Review Beyond Defined Scope

The `pr_reviewer` skill checks for:
- Unused includes in the generated package
- Missing includes (not reachable except transitively)
- Duplicate package assignments

It does **not** perform:
- General code quality review
- Style enforcement
- Security scanning
- Performance analysis

---

## Why These Are Out of Scope

These exclusions exist because:

1. **Scope control:** The migration is complex enough without adding VOB-to-Git migration, which is a separate multi-month project.
2. **Safety:** General refactoring touches production code and requires extensive testing beyond what this agent provides.
3. **Tooling:** Some of these tasks (header reorganization, history replay) require specialized tools the team already has or has deliberately decided not to use.
4. **Separation of concerns:** The agent's value is in the package assignment and build file generation. Adding scope risks diluting that value.

If the team decides to bring any of these capabilities in scope in the future, the spec must be updated first, followed by the constitution, then the agent implementation.
