# Agents — Two-Agent Architecture

This document specifies the two-agent architecture of the ClearCase Migration Agent. Both agents share the same codebase but have distinct modes, permissions, and responsibilities.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Engineer (VS Code)                        │
│                                                             │
│  @agent plan --view-path /vobs/product --release R2024.1   │
│         │                                                   │
│         ▼                                                   │
│  ┌─────────────────────┐    Plan PR    ┌──────────────────┐ │
│  │   Planning Agent    │──────────────▶│  Workspace Repo  │ │
│  │   (read-only)       │               │  (GitHub PR)     │ │
│  └─────────────────────┘               └──────────────────┘ │
│                                               │             │
│                                         Human review        │
│                                         & approval          │
│                                               │             │
│  @agent implement --plan-approved             │             │
│             --enable-writes                   │             │
│             --package libfoo    ◀─────────────┘             │
│         │                                                   │
│         ▼                                                   │
│  ┌─────────────────────┐   Package PR   ┌─────────────────┐ │
│  │ Implementation Agent│───────────────▶│  Package Repo   │ │
│  │   (read-write)      │                │  (CI gate)      │ │
│  └─────────────────────┘                └─────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## Planning Agent

### Identity

- **Name:** `planning-agent`
- **Mode:** Read-only + Interactive
- **Invocation:** `@agent plan [flags]`

### Purpose

Settle the complete package partition before any building starts. Produce a human-reviewable plan as a PR.

### Flags

| Flag | Required | Description |
|---|---|---|
| `--view-path <path>` | Yes | Absolute path to mounted ClearCase view |
| `--release <id>` | Yes | Product/release identifier (e.g., `R2024.1`) |
| `--product <name>` | No | Limit scope to a specific product (default: all products with build records) |
| `--reuse-run <runId>` | No | Resume from a previous planning run (reuse ingested data) |

### Execution Flow

```
1. Load existing packages and locked packages (existing_package_loader)
2. Ingest build records for scope (build_record_ingester)
3. Build include graph with macro context (fw_include_graph)
4. Derive macro matrix / Conan option candidates (macro_matrix)
5. Run header pruner (header_pruner)
6. Propose package assignment diff (assignment logic)
7. Score per-file risk (risk_scoring)
8. Detect circular dependencies; propose resolutions (DependencyGraph output)
9. Conduct interactive Q&A for open items (user_dialog)
10. Validate all artifacts against schemas
11. Write run directory (artifacts/runs/{runId}/)
12. Open plan PR to workspace repo (pr_writer — read-only PR creation)
```

### Skills Used (Planning Agent)

| Skill | Spec |
|---|---|
| `build_record_ingester` | `skills/build-record-ingester.md` |
| `existing_package_loader` | `skills/existing-package-loader.md` |
| `fw_include_graph` | `skills/fw-include-graph.md` |
| `macro_matrix` | `skills/macro-matrix.md` |
| `header_pruner` | `skills/header-pruner.md` |
| `risk_scoring` | *(spec TBD — see `open-questions.md §9`)* |
| `user_dialog` | *(built-in Copilot Chat interaction)* |

### Output Artifacts

| Artifact | Schema | Description |
|---|---|---|
| `PackageAssignment.json` | `PackageAssignment.schema.json` | File-to-package mapping diff |
| `DependencyGraph.json` | `DependencyGraph.schema.json` | Package-level DAG + SCCs |
| `CopyMapProposal.json` | `CopyMapProposal.schema.json` | Copy-map entries per package |
| `MacroMatrix.json` | `MacroMatrix.schema.json` | Conan option candidates per package |
| `UserDecisions.json` | `UserDecisions.schema.json` | Q&A capture |
| `Manifest.json` | `Manifest.schema.json` | Run metadata |
| `RunLogEvent.jsonl` | `RunLogEvent.schema.json` | Skill invocation audit log |

### Hard Constraints (Planning Agent)

- MUST NOT write, modify, or delete any source file
- MUST NOT create branches in the source VOB
- MUST NOT push commits to the source repository
- MUST load `existing_package_loader` before proposing any assignments
- MUST emit only a diff (not a from-scratch partition)
- MUST wait for user decision on circular dependencies before proceeding
- MUST validate all artifacts before writing to run directory

---

## Implementation Agent

### Identity

- **Name:** `implementation-agent`
- **Mode:** Read-write (requires explicit flags)
- **Invocation:** `@agent implement --plan-approved --enable-writes --package <name>`

### Purpose

Generate build files for one package per invocation. Open a PR. Wait for CI. Move to next package.

### Flags

| Flag | Required | Description |
|---|---|---|
| `--plan-approved` | Yes | Indicates the plan PR has been approved |
| `--enable-writes` | Yes | Enables file writes, commits, and PRs |
| `--package <name>` | Yes | The package to implement |
| `--run-id <runId>` | No | Which planning run to use (default: most recent approved run) |
| `--fix-ci` | No | Diagnose current CI failure and propose fix (skips generation, runs `compiler_error_tracer`) |
| `--review-pr <prNumber>` | No | Run `pr_reviewer` on the specified PR |

### Preconditions

Before any write operation, the Implementation Agent MUST verify:

1. `--plan-approved` flag is set
2. `--enable-writes` flag is set
3. `Manifest.json` for the referenced run has `"status": "approved"`
4. `PackageAssignment.json` contains the target package with `"status": "accepted"`
5. The target package is not locked

If any precondition fails, the agent halts and reports the violation.

### Execution Flow (normal run)

```
1. Validate preconditions (see above)
2. Load PackageAssignment.json, MacroMatrix.json for target package
3. Generate copy-map entries (copy_map_generator)
4. Generate CMakeLists.txt (cmake_generator)
5. Generate conanfile.py with derived options (conan_generator)
6. Generate Conan patches if needed (patch_proposer)
7. Write all generated files to staging area
8. Open PR to package repo (pr_writer)
9. Run pr_reviewer on the new PR
10. Log all actions to ImplementationLog.json
11. Wait for CI result
12. On CI failure: invoke compiler_error_tracer, log to BuildErrorReport.json
```

### Execution Flow (`--fix-ci` mode)

```
1. Read CI build log from package repo
2. Parse and classify errors (compiler_error_tracer)
3. Propose fixes (copy-map changes or Conan option changes)
4. Log to BuildErrorReport.json
5. If fix requires a code patch: emit PatchProposal.json for human review
6. Do not auto-apply patches — always require human approval
```

### Skills Used (Implementation Agent)

| Skill | Spec |
|---|---|
| `copy_map_generator` | `skills/copy-map-generator.md` |
| `cmake_generator` | *(no separate spec — driven by `PackageAssignment.json` and `MacroMatrix.json`)* |
| `conan_generator` | *(no separate spec — driven by `MacroMatrix.json`)* |
| `patch_proposer` | *(invoked by `dependency_breaker` and `compiler_error_tracer`)* |
| `dependency_breaker` | `skills/dependency-breaker.md` |
| `compiler_error_tracer` | `skills/compiler-error-tracer.md` |
| `pr_writer` | *(built-in GitHub API wrapper)* |
| `pr_reviewer` | `skills/pr-reviewer.md` |

### Output Artifacts

| Artifact | Schema | Description |
|---|---|---|
| `ImplementationLog.json` | `ImplementationLog.schema.json` | Log of generated files |
| `BuildErrorReport.json` | `BuildErrorReport.schema.json` | CI failure analysis (when applicable) |
| `PatchProposal.json` | `PatchProposal.schema.json` | Proposed code patches (when applicable) |
| `Manifest.json` | `Manifest.schema.json` | Run metadata |
| `RunLogEvent.jsonl` | `RunLogEvent.schema.json` | Skill invocation audit log |

### Hard Constraints (Implementation Agent)

- MUST NOT execute without both `--plan-approved` and `--enable-writes`
- MUST NOT implement a package with `"status": "proposed"` or `"status": "conflict"`
- MUST NOT touch locked packages
- MUST NOT add Conan options not listed in `MacroMatrix.json` for the package
- MUST NOT bundle multiple packages in one PR
- MUST NOT open the next package PR before current package CI passes
- MUST validate all artifacts before writing to run directory

---

## Shared Infrastructure

Both agents share:

- **Run directory convention:** `artifacts/runs/{runId}/`
- **Schema validation:** All artifacts validated against `specs/schemas/`
- **`RunLogEvent.jsonl`:** Appended by every skill invocation
- **`Manifest.json`:** Written at start (status: `running`) and updated at end (status: `complete`/`error`/`approved`)

---

## Agent Isolation

The two agents have **separate contexts**. The Implementation Agent reads artifacts produced by the Planning Agent but does not share memory or state with it at runtime. This ensures:

- The implementation cannot accidentally modify planning data
- Each run is independently auditable via `Manifest.json` and `RunLogEvent.jsonl`
- Planning can be re-run (to update the prior) without restarting implementation
