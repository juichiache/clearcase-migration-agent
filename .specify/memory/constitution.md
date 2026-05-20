# Constitution — ClearCase Migration Intelligence Layer

> Source of truth: `specs/functional/overview.md` and child specs in `specs/functional/`.

---

## Core Rules

1. **READ-ONLY by default** — No file writes, commits, or pushes without explicit `--enable-writes` flag.
2. **Plan before building** — Package assignments must be fully accepted before any implementation skill runs.
3. **Writes require explicit gate** — Both `--enable-writes` AND `--plan-approved` must be set; either alone is insufficient.
4. **Artifacts must validate schemas** — Every artifact is validated against its JSON Schema in `specs/schemas/` before persistence; CI fails on schema violations.
5. **Produce diffs, not from-scratch partitions** — Agent always starts from the existing package state and emits only changes.
6. **Locked packages are immutable** — Files or packages flagged `"locked": true` may never be reassigned or modified.
7. **Conan options are derived, not invented** — All `options` in generated `conanfile.py` must trace to observed macro combinations in `MacroMatrix.json`.
8. **One package per PR** — Implementation PRs are scoped to exactly one package; no bundling.
9. **CI gate between packages** — The next package's PR is not opened until the current package's CI passes.
10. **Tool execution logged** — Every skill invocation is logged with inputs, outputs, and timing to `RunLogEvent.jsonl`.
11. **Two-agent isolation** — Planning Agent and Implementation Agent have separate contexts and separate run directories.
12. **Planning Agent is read-only** — Cannot modify source code, create branches, or push commits.
13. **Implementation Agent requires approval** — Cannot execute without approved Planning artifacts and both gate flags.
14. **Reason field mandatory** — Every proposed package assignment change must include a `reason` field citing the source evidence.
15. **Open questions are first-class** — Unresolved items are tracked in `specs/functional/open-questions.md`; agents must not guess at answers and must reference the relevant open question.

---

## Agent Rules

### Planning Agent
- **Mode:** Read-only + Interactive
- **Can:** Read ClearCase view, ingest build records, build include graph, run macro matrix, propose assignments as diff, ask questions via `user_dialog`, emit plan PR
- **Cannot:** Modify any source file, create branches, push commits, run implementation skills

### Implementation Agent
- **Mode:** Read-write (after approval)
- **Can:** Generate CMakeLists.txt, conanfile.py, copy-map entries, patches; open PRs; diagnose CI failures
- **Cannot:** Execute without `--plan-approved` + `--enable-writes`; touch locked packages; add uninvented Conan options; bundle multiple packages in one PR

---

## Schema Enforcement

All artifacts are defined by JSON Schema draft-07 files in `specs/schemas/`. The following table maps artifact to schema:

| Artifact | Schema File |
|---|---|
| `PackageAssignment.json` | `specs/schemas/PackageAssignment.schema.json` |
| `DependencyGraph.json` | `specs/schemas/DependencyGraph.schema.json` |
| `CopyMapProposal.json` | `specs/schemas/CopyMapProposal.schema.json` |
| `MacroMatrix.json` | `specs/schemas/MacroMatrix.schema.json` |
| `UserDecisions.json` | `specs/schemas/UserDecisions.schema.json` |
| `ImplementationLog.json` | `specs/schemas/ImplementationLog.schema.json` |
| `BuildErrorReport.json` | `specs/schemas/BuildErrorReport.schema.json` |
| `PatchProposal.json` | `specs/schemas/PatchProposal.schema.json` |
| `Manifest.json` | `specs/schemas/Manifest.schema.json` |
| `RunLogEvent.jsonl` | `specs/schemas/RunLogEvent.schema.json` (per line) |

---

## Run Directory Convention

Each agent run produces artifacts in a timestamped directory:

```
artifacts/
└── runs/
    └── {runId}/            # e.g., plan-20241215T143022Z
        ├── Manifest.json
        ├── PackageAssignment.json
        ├── DependencyGraph.json
        ├── CopyMapProposal.json
        ├── MacroMatrix.json
        ├── UserDecisions.json
        └── RunLogEvent.jsonl
```

The `runId` is recorded in `Manifest.json`. Implementation runs reference the planning `runId` in their manifest.

---

## Approval Protocol

1. Planning Agent emits plan artifacts to run directory.
2. Planning Agent opens a PR to the workspace repo with all planning artifacts.
3. One or more designated approvers (see `open-questions.md §7.4`) review the PR.
4. On approval, the PR is merged, and the `--plan-approved` flag may be passed to the Implementation Agent.
5. Implementation Agent verifies the manifest status is `approved` before proceeding.

---

## Data Handling

- **ClearCase view paths** are used at runtime but not persisted in artifacts pushed to GitHub. Use relative paths or placeholders.
- **Internal product names** in artifacts: use the product identifier as defined in `Manifest.json`; do not embed additional internal metadata.
- **Build record YAML** is read at runtime; it is not committed to this repository.
- **Source files** are never copied into this repository; only file paths (relative to view root) appear in artifacts.

---

## Status of Implementation

This is the initial spec kit. No skills are implemented yet. The following tracks spec vs. implementation status:

| Component | Spec | Implementation |
|---|---|---|
| Planning Agent shell | ✅ Specified | ❌ Not started |
| Implementation Agent shell | ✅ Specified | ❌ Not started |
| `build_record_ingester` | ✅ Specified | ❌ Not started |
| `existing_package_loader` | ✅ Specified | ❌ Not started |
| `fw_include_graph` | ✅ Specified | ❌ Not started |
| `macro_matrix` | ✅ Specified | ❌ Not started |
| `header_pruner` | ✅ Specified | ❌ Not started |
| `dependency_breaker` | ✅ Specified | ❌ Not started |
| `copy_map_generator` | ✅ Specified | ❌ Not started |
| `compiler_error_tracer` | ✅ Specified | ❌ Not started |
| `pr_reviewer` | ✅ Specified | ❌ Not started |
| JSON Schema validation | ✅ Schemas defined | ❌ Not wired into CI |
| Plan PR workflow | ✅ Specified | ❌ Not started |
