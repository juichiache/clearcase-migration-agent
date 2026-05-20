# ClearCase → CMake/Conan Migration Agent

A two-phase AI agent that helps engineers migrate a massive ClearCase codebase to CMake + Conan. The agent is spec-driven: every behavior is documented in `specs/`, every output validates against a JSON Schema, and every write requires explicit human approval.

---

## Quick Start

### Prerequisites

- VS Code with GitHub Copilot Chat extension
- Remote-SSH extension (to connect to a machine with a ClearCase view mounted)
- Node.js 20+
- A ClearCase view already mounted at a known path

### Invocation

1. SSH into the runner machine via VS Code Remote-SSH.
2. Open this repository in VS Code.
3. Open GitHub Copilot Chat.
4. Start the **Planning Agent**:

```
@agent plan --view-path /vobs/myproduct --release R2024.1
```

The agent will ingest build records, load existing packages, build an include graph, and propose package assignments as a diff. It emits a PR to the workspace repo for human review.

5. After reviewing and approving the plan PR, start the **Implementation Agent**:

```
@agent implement --plan-approved --enable-writes --package libfoo
```

The agent generates `CMakeLists.txt`, `conanfile.py`, copy-map entries, and opens a PR. CI must pass before the next package is started.

---

## Project Structure

```
clearcase-migration-agent/
├── AGENTS.md                        # Custom instructions for agent runtime
├── README.md                        # This file
├── .gitignore
├── .specify/
│   └── memory/
│       └── constitution.md          # Core governance rules
├── specs/
│   ├── functional/
│   │   ├── overview.md              # High-level design (§1–7)
│   │   ├── agents.md                # Two-agent architecture
│   │   ├── artifacts.md             # Artifact definitions and lifecycle
│   │   ├── operating-model.md       # Day-to-day usage
│   │   ├── testing.md               # 4-level validation plan
│   │   ├── out-of-scope.md          # Explicit exclusions
│   │   ├── open-questions.md        # Unresolved items from design doc
│   │   └── skills/
│   │       ├── build-record-ingester.md
│   │       ├── existing-package-loader.md
│   │       ├── fw-include-graph.md
│   │       ├── macro-matrix.md
│   │       ├── header-pruner.md
│   │       ├── dependency-breaker.md
│   │       ├── copy-map-generator.md
│   │       ├── compiler-error-tracer.md
│   │       └── pr-reviewer.md
│   └── schemas/
│       ├── PackageAssignment.schema.json
│       ├── DependencyGraph.schema.json
│       ├── CopyMapProposal.schema.json
│       ├── MacroMatrix.schema.json
│       ├── UserDecisions.schema.json
│       ├── ImplementationLog.schema.json
│       ├── BuildErrorReport.schema.json
│       ├── PatchProposal.schema.json
│       ├── Manifest.schema.json
│       └── RunLogEvent.schema.json
└── artifacts/                       # Runtime artifact output (git-ignored except .gitkeep)
```

---

## How the Spec Kit Works

This project follows a **spec-driven** methodology:

1. **Specs are the contract** — Every skill, artifact, and agent behavior is specified before code is written. Developers and AI alike implement from the spec.
2. **Schemas are enforced** — Every artifact produced by the agent is validated against its JSON Schema in `specs/schemas/`. CI fails on schema violations.
3. **Open questions are first-class** — Items marked `[OPEN]` in specs mean "we know we don't know this yet." They are tracked in `specs/functional/open-questions.md`.
4. **The constitution governs** — `.specify/memory/constitution.md` defines the non-negotiable rules that all agents must follow.

---

## Two-Phase Architecture

### Phase 1 — Planning Agent (read-only)

Settles package assignments before any building starts. The agent:
- Loads existing packages and annotated copy-maps as a weighted prior
- Ingests scripted YAML build records (macros, per-source headers, library file sets)
- Proposes package assignments as a *diff* against the current state (not from scratch)
- Detects circular dependencies and proposes ranked resolutions
- Generates macro matrices for Conan options
- **Output:** A PR to the workspace repo containing all planning artifacts for human review

### Phase 2 — Implementation Agent (read-write, requires approval)

Executes one package per PR, with CI gate between each:
- Generates `CMakeLists.txt`, `conanfile.py`, copy-map entries, Conan patches
- Reads build errors and proposes fixes (`compiler_error_tracer`)
- Reviews PRs for include issues (`pr_reviewer`)

---

## Key Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Source of truth | Scripted YAML build records | Not the folder tree — YAML captures macros and header sets per source file |
| Existing packages | Weighted prior (not discarded) | Agent proposes a diff; locked packages are never touched |
| Conan options | Derived from observed macro combos | Never invented; derived from `macro_matrix` skill output |
| Build order | Package assignment fully complete before building | Avoids mid-build repartitioning |
| Invocation | VS Code + Remote-SSH + GitHub Copilot Chat | Agent runs on runner with ClearCase view mounted |
| Plan review | PR to workspace repo | Human engineers review and approve before implementation |

---

## For New Engineers

1. Read `specs/functional/overview.md` for the design summary.
2. Read `specs/functional/agents.md` for the two-agent architecture.
3. Read `AGENTS.md` for the rules that govern agent behavior.
4. Read `.specify/memory/constitution.md` for the governance constitution.
5. Check `specs/functional/open-questions.md` for items that need team input.
6. When implementing a skill, start from its spec in `specs/functional/skills/`.

---

## Contributing

- Do not implement a skill without a spec in `specs/functional/skills/`.
- Do not produce an artifact without a schema in `specs/schemas/`.
- Open questions go in `specs/functional/open-questions.md`, not in code comments.
- All agent writes require the `--plan-approved` and `--enable-writes` flags.
