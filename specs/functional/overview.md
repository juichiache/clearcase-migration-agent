# Overview — ClearCase → CMake/Conan Migration Agent

> Covers design document sections 1–7. For the full skill specifications, see `skills/`. For open questions, see `open-questions.md`.

---

## §1 — Context and Scope

### Problem

The codebase being migrated is a large, multi-product ClearCase repository. It is structured as a set of VOBs containing C/C++ source files, and has been built historically with an omake-based build system. The migration target is CMake + Conan.

The migration is not a greenfield rewrite: thousands of source files must be partitioned into Conan packages. Many files are already assigned to packages (the existing partition). The agent's job is to help complete and validate this partition, then generate the build files.

**Key constraints:**
- Some packages are already migrated ("locked"). The agent must never reassign their files.
- The build system is complex: macros control which headers are included per compilation, and Conan options must reflect these macro combinations.
- The codebase is too large for any individual engineer to hold in their head; the agent provides structure and automation.

> [OPEN] §1.1 — Which product/release is used as the primary test fixture? See `open-questions.md §1.1`.
> [OPEN] §1.2 — Which already-migrated product serves as the correctness answer key? See `open-questions.md §1.2`.
> [OPEN] §1.3 — Are there known circular dependencies in the existing codebase? See `open-questions.md §1.3`.
> [OPEN] §1.4 — Are there known files that appear in multiple products? See `open-questions.md §1.4`.

---

## §2 — Build Records as Source of Truth

Build records are YAML files produced by a Python extraction script. Each build record captures, for a specific product and release:

- The set of source files compiled
- The macro combinations active for each source file
- The headers included (per-source, under each macro combination)
- The library file sets linked

**Why build records, not the folder tree?**

The folder tree does not encode which files are actually compiled, which macros are active, or which headers are reachable. Build records contain this information directly. They are the authoritative source of fact about what gets compiled and how.

The `build_record_ingester` skill reads these YAML files and produces normalized, queryable in-memory representations.

> [OPEN] §2.1 — What is the exact YAML schema produced by the Python extraction script? This is CRITICAL for implementing `build_record_ingester`. See `open-questions.md §2.1`.
> [OPEN] §2.2 — Which products/releases have scripted YAML build records today? Which are in progress? See `open-questions.md §2.2`.
> [OPEN] §2.3 — Should the agent also parse omake makefiles as a fallback? See `open-questions.md §2.3`.

---

## §3 — Planning Phase

The Planning Agent settles the complete package partition before any building starts. It operates entirely in read-only mode.

### §3.1 Copy-Maps as the Prior

The existing package partition is encoded in annotated copy-maps. The `existing_package_loader` skill reads these and treats them as a weighted prior. Files already assigned to a locked package are immutable.

The agent proposes a *diff*: only files with no assignment, conflicting assignments, or new evidence from build records will have proposed changes. Files with confident existing assignments are left as-is.

> [OPEN] §3.1 — What is the exact copy-map file format? A real example is needed. This is CRITICAL for `copy_map_generator` output format. See `open-questions.md §3.1`.
> [OPEN] §3.3 — How strong is the default weight for existing package assignments? What confidence bar must the agent clear before proposing a change? See `open-questions.md §3.3`.

### §3.2 Include Graph

The `fw_include_graph` skill builds a directed graph where:
- Nodes are source files and header files
- Edges carry: (includer, includee, macro_set)

The macro_set on an edge captures the macro combination under which the include is active. This is essential for correctly determining package dependencies.

### §3.3 Macro Matrix

The `macro_matrix` skill analyzes all per-file macro combinations across the package and produces the minimal set of Conan options that covers all observed combinations. This is the primary time-saving feature: manual derivation of Conan options from macros is error-prone and tedious.

**Rule:** No Conan option may appear in a generated `conanfile.py` unless it appears in the package's entry in `MacroMatrix.json`.

> [OPEN] §3.2 — Should the agent generate Conan patches, or leave that entirely to humans? See `open-questions.md §3.2`.
> [OPEN] §3.5 — Which CMake generator target: GNU make, Ninja, or Visual Studio? See `open-questions.md §3.5`.

### §3.4 Circular Dependency Detection

The `fw_include_graph` / `DependencyGraph.json` output identifies package-level cycles using Tarjan's SCC algorithm. For each SCC:

1. The agent reports the cycle members and the specific include edges that form the cycle.
2. The agent proposes ranked resolution strategies (e.g., extract interface header, introduce indirection, merge packages).
3. The agent does NOT auto-resolve. It waits for user decision via `user_dialog`.

> [OPEN] §5.4 — Resolution ranking algorithm and whether any class of cycles can be auto-resolved. See `open-questions.md §5.4`.

### §3.5 Package Proposal Output

The Planning Agent produces:

- `PackageAssignment.json` — the proposed partition, as a diff
- `DependencyGraph.json` — the package-level dependency graph
- `CopyMapProposal.json` — copy-map entries ready for review
- `MacroMatrix.json` — Conan option candidates per package
- `UserDecisions.json` — all interactive Q&A from the planning session
- `Manifest.json` — run metadata
- `RunLogEvent.jsonl` — audit log

These artifacts are emitted as a PR to the workspace repo for human review and approval.

> [OPEN] §3.4 — Should the agent use senior engineer high-level breakdowns as an additional input? See `open-questions.md §3.4`.

---

## §4 — Implementation Phase

After the plan PR is approved, the Implementation Agent generates build files one package at a time.

**Per package, the agent generates:**
- `CMakeLists.txt`
- `conanfile.py` (with options derived from `MacroMatrix.json`)
- Copy-map entries (in the existing format)
- Conan patches (for files that won't build clean without source changes)

**CI gate:** Each package PR must pass CI before the next package PR is opened.

**On CI failure:** The `compiler_error_tracer` skill reads the build log, diagnoses the failure, and proposes fixes as copy-map changes or Conan option adjustments.

> [OPEN] §4.1 — Team-to-folder mapping for PR routing. See `open-questions.md §4.1`.
> [OPEN] §6.2 — Confirm one-package-per-PR with CI gate as the team's workflow preference. See `open-questions.md §6.2`.

---

## §5 — File Assignment Edge Cases

### §5.1 Forkable Files

Some files may legitimately belong to multiple packages (e.g., shared utility headers included by both packages and stable platform code).

> [OPEN] §5.1 — Which files/patterns are considered "forkable"? See `open-questions.md §5.1`.

### §5.2 Orphan Files

Files present in the ClearCase view but absent from all build records.

> [OPEN] §5.2 — How to handle orphan files: (a) ignore, (b) flag in report, (c) assign by include proximity? See `open-questions.md §5.2`.

### §5.3 Ambiguous Header Resolution

A header included under macro set A resolves to `/vobs/product/include/foo.h`, but under macro set B resolves to `/vobs/platform/include/foo.h`.

> [OPEN] §5.3 — Resolution strategy: (a) pick one + flag, (b) refuse the assignment, (c) model both in the graph? See `open-questions.md §5.3`.

---

## §6 — Design Decisions to Confirm

The following decisions appear in the design document as intended but require team confirmation:

> [OPEN] §6.1 — Confirm build-records-as-source-of-truth (not folder tree). See `open-questions.md §6.1`.
> [OPEN] §6.2 — Confirm one-package-per-PR with CI gate. See `open-questions.md §6.2`.
> [OPEN] §6.3 — Confirm Conan options derived-not-invented. See `open-questions.md §6.3`.
> [OPEN] §6.4 — Confirm package assignment fully before building. See `open-questions.md §6.4`.

---

## §7 — Operating Environment

### §7.1 Invocation

The agent runs on a machine that has a ClearCase view mounted. Engineers invoke the agent via VS Code + Remote-SSH + GitHub Copilot Chat. The agent is not a cloud service; it runs in the engineer's VS Code session.

> [OPEN] §7.1 — Confirm VS Code + Remote-SSH + Copilot Chat as the invocation model. See `open-questions.md §7.1`.

### §7.2 Plan Review

The Planning Agent emits its proposed plan as a PR to a designated workspace repo. Team members review the PR, comment on specific package assignments, and approve or request changes.

> [OPEN] §7.2 — Confirm PR-based plan review as the workflow. Identify the workspace repo. See `open-questions.md §7.2`.

### §7.3 PR Review Capabilities

The `pr_reviewer` skill operates in two modes:
1. **Include audit** — checks each PR for unused includes and missing includes
2. **Duplicate assignment audit** — checks for files assigned to multiple packages (unless marked forkable)

> [OPEN] §7.3 — Confirm both PR reviewer capabilities are needed. See `open-questions.md §7.3`.

### §7.4 Approval Authority

> [OPEN] §7.4 — Who has final approval authority for plan PRs? Is this per-team or centralized? See `open-questions.md §7.4`.
