# Artifacts — Definitions, Schemas, Lifecycle, and Approval Protocol

All artifacts produced by the migration agent are JSON or JSONL files that validate against JSON Schema draft-07 schemas in `specs/schemas/`. This document defines the lifecycle, purpose, and approval protocol for each artifact.

---

## Artifact Lifecycle

```
Planning Run                     Implementation Run
─────────────                    ─────────────────
build_record_ingester            (reads planning artifacts)
existing_package_loader                │
fw_include_graph                       │
macro_matrix                           ▼
header_pruner           ──────▶ PackageAssignment.json (accepted)
assignment logic                       │
risk_scoring                           ▼
user_dialog                    copy_map_generator ──▶ CopyMapProposal
        │                      cmake_generator    ──▶ CMakeLists.txt
        ▼                      conan_generator    ──▶ conanfile.py
PackageAssignment.json         compiler_error_tracer (on CI fail)
DependencyGraph.json           pr_reviewer
CopyMapProposal.json                   │
MacroMatrix.json                       ▼
UserDecisions.json             ImplementationLog.json
Manifest.json                  BuildErrorReport.json (conditional)
RunLogEvent.jsonl              PatchProposal.json (conditional)
        │                      Manifest.json
        ▼                      RunLogEvent.jsonl
  Plan PR (workspace repo)
        │
   Human review
        │
        ▼
  --plan-approved flag
```

---

## Planning Agent Artifacts

### 1. `PackageAssignment.json`

**Schema:** `specs/schemas/PackageAssignment.schema.json`

**Purpose:** The central output of the Planning Agent. Maps each source file to a package. This is a diff against the existing partition.

**Key fields:**
- `runId` — links to `Manifest.json`
- `assignments[]` — one entry per file
  - `filePath` — path relative to ClearCase view root
  - `package` — assigned package name
  - `status` — `proposed` | `accepted` | `locked` | `conflict`
  - `confidence` — 0.0–1.0 confidence score
  - `reason` — human-readable reason for any change from existing assignment
  - `previousPackage` — the existing assignment (null if unassigned)
  - `evidence` — list of build record / include edge sources

**Lifecycle:**
1. Written by Planning Agent (status: `proposed`)
2. Reviewed in plan PR; team may annotate with accepted/conflict
3. Implementation Agent reads only `accepted` entries
4. `locked` entries are never changed

**Approval requirement:** Human review of the plan PR. Engineer must resolve `conflict` entries before implementation can start.

---

### 2. `DependencyGraph.json`

**Schema:** `specs/schemas/DependencyGraph.schema.json`

**Purpose:** Package-level directed dependency graph. Identifies circular dependencies (SCCs).

**Key fields:**
- `packages[]` — nodes in the graph
- `edges[]` — directed edges (from → to), with edge strength and macro context
- `sccs[]` — strongly connected components (cycles); each has:
  - `members[]` — package names in the cycle
  - `cycleEdges[]` — the specific include edges that form the cycle
  - `proposedResolutions[]` — ranked resolution strategies

**Lifecycle:**
1. Written by Planning Agent
2. Included in plan PR for review
3. Circular dependencies must be resolved before implementation

---

### 3. `CopyMapProposal.json`

**Schema:** `specs/schemas/CopyMapProposal.schema.json`

**Purpose:** Copy-map entries per package in the existing copy-map format, ready to be merged into the team's actual copy-maps.

> [OPEN] §3.1 — The exact copy-map format is not yet defined. See `open-questions.md §3.1`. This schema will be updated when the format is confirmed.

**Key fields:**
- `packages[]` — one entry per package
  - `packageName`
  - `entries[]` — copy-map entries (format TBD)
  - `status` — `proposed` | `accepted` | `locked`

**Lifecycle:**
1. Written by Planning Agent
2. Reviewed in plan PR
3. On approval, entries are merged into the actual copy-map files by the Implementation Agent

---

### 4. `MacroMatrix.json`

**Schema:** `specs/schemas/MacroMatrix.schema.json`

**Purpose:** Per-package mapping of observed macro combinations to source files. This is the source of truth for Conan options.

**Key fields:**
- `packages[]` — one entry per package
  - `packageName`
  - `combinations[]` — each is a distinct macro set observed in build records
    - `macros` — `{name: value}` map
    - `sourceFiles[]` — files that use this combination
    - `conanOptionName` — derived option name for this combination
    - `conanOptionValue` — the option value

**Lifecycle:**
1. Written by Planning Agent (macro_matrix skill)
2. Read by Implementation Agent (conan_generator)
3. Defines the complete allowed set of Conan options for each package

**Critical rule:** No option may appear in a generated `conanfile.py` unless it is listed here for that package.

---

### 5. `UserDecisions.json`

**Schema:** `specs/schemas/UserDecisions.schema.json`

**Purpose:** Records all Q&A interactions from the planning session. Provides an audit trail for decisions that shaped the package partition.

**Key fields:**
- `decisions[]` — one per question asked
  - `id` — unique decision identifier
  - `question` — the question asked by the agent
  - `answer` — the engineer's response
  - `timestamp`
  - `sectionReference` — which open-question or spec section prompted the question
  - `impactedAssignments[]` — which `PackageAssignment` entries this decision affects

**Lifecycle:**
1. Appended incrementally during Planning Agent session
2. Included in plan PR for review
3. Provides traceable reasoning for package assignment decisions

---

## Implementation Agent Artifacts

### 6. `ImplementationLog.json`

**Schema:** `specs/schemas/ImplementationLog.schema.json`

**Purpose:** Records all files generated by the Implementation Agent, linked to package names and planning run tasks.

**Key fields:**
- `runId` — this implementation run
- `planRunId` — the planning run this implements
- `packages[]` — one entry per implemented package
  - `packageName`
  - `planTaskRef` — reference to the PackageAssignment entry
  - `generatedFiles[]` — list of generated file paths with their type (cmake/conan/copymap/patch)
  - `prUrl` — the PR opened for this package
  - `ciStatus` — `pending` | `passing` | `failing`

**Lifecycle:**
1. Written by Implementation Agent as packages are processed
2. Updated with CI status as results arrive

---

### 7. `BuildErrorReport.json`

**Schema:** `specs/schemas/BuildErrorReport.schema.json`

**Purpose:** Records CI build failures with proposed fixes. Produced by `compiler_error_tracer`.

**Key fields:**
- `packageName`
- `prNumber`
- `buildLogUrl`
- `errors[]` — one per diagnosed error
  - `errorType` — `missing_include` | `wrong_package` | `undefined_symbol` | `missing_option` | `other`
  - `location` — file path and line number
  - `rawMessage` — original compiler error text
  - `diagnosis` — agent's interpretation
  - `proposedFix` — `{type, description, changeRef}` where `type` is `copy_map_change`, `option_change`, or `code_patch`

**Lifecycle:**
1. Written on CI failure
2. Reviewed by engineer
3. Approved fixes applied via follow-up PR or patch proposal

---

### 8. `PatchProposal.json`

**Schema:** `specs/schemas/PatchProposal.schema.json`

**Purpose:** Proposed code patches. These are narrow, dependency-breaking edits and Conan patches — not general refactoring. Always require human approval before application.

**Key fields:**
- `proposals[]` — one per proposed patch
  - `id` — unique patch identifier
  - `patchType` — `dependency_break` | `conan_patch`
  - `targetFile` — file to be patched
  - `rationale` — why this patch is needed
  - `unifiedDiff` — the patch in unified diff format
  - `riskLevel` — `low` | `medium` | `high`
  - `approvalStatus` — `pending` | `approved` | `rejected`

**Lifecycle:**
1. Proposed by `dependency_breaker` or `compiler_error_tracer`
2. Reviewed and approved/rejected by engineer
3. Approved patches applied by Implementation Agent via PR

---

## Shared Artifacts

### 9. `Manifest.json`

**Schema:** `specs/schemas/Manifest.schema.json`

**Purpose:** Run metadata for both Planning and Implementation runs. Links all artifacts for a run.

**Key fields:**
- `runId` — unique identifier for this run (e.g., `plan-20241215T143022Z`)
- `agentType` — `planning` | `implementation`
- `timestamp` — ISO 8601 start time
- `status` — `running` | `complete` | `error` | `approved`
- `product` — product identifier
- `release` — release identifier
- `viewPath` — ClearCase view path (runtime only; may be redacted in artifact)
- `artifactPaths` — map of artifact type to file path
- `planRunId` — (implementation runs only) the planning run this implements

**Lifecycle:**
1. Written at run start (status: `running`)
2. Updated at run end (status: `complete` or `error`)
3. Updated by approver when plan PR merged (status: `approved`)

---

### 10. `RunLogEvent.jsonl`

**Schema:** `specs/schemas/RunLogEvent.schema.json` (per line)

**Purpose:** Audit log of every skill invocation. JSONL format (one JSON object per line).

**Key fields per line:**
- `timestamp` — ISO 8601
- `runId`
- `skill` — skill name
- `phase` — `start` | `complete` | `error`
- `inputs` — summary of inputs (not full data)
- `outputs` — summary of outputs (not full data)
- `durationMs` — elapsed time (on completion)
- `error` — error message if phase is `error`

**Lifecycle:**
1. Appended by every skill invocation
2. Never overwritten — append-only
3. Included in run directory

---

## Approval Protocol

### Plan Approval

1. Planning Agent completes and writes all artifacts to `artifacts/runs/{runId}/`.
2. Planning Agent opens a PR to the workspace repo with:
   - All planning artifacts as files
   - A summary comment explaining proposed changes, conflicts, and circular dependencies
3. Engineers review the PR. They may:
   - Approve the full plan
   - Annotate specific assignments as `conflict` (requiring resolution)
   - Request changes (agent re-runs affected skills)
4. On PR merge, `Manifest.json` status is updated to `approved`.
5. The `--plan-approved` flag may now be passed to the Implementation Agent.

### Patch Approval

1. Implementation Agent generates `PatchProposal.json`.
2. Patch proposals are included in the relevant package PR as a comment.
3. Engineers review each patch individually.
4. Only approved patches (field `approvalStatus: "approved"`) are applied.
5. Rejected patches are logged; the agent proposes an alternative or escalates.
