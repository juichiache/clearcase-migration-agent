# Testing — 4-Level Validation Plan

This document defines the four levels of validation used to verify the migration agent produces correct results. Levels are cumulative: a higher level subsumes lower levels.

---

## Overview

| Level | Name | What it validates | When it runs |
|---|---|---|---|
| 1 | Schema validation | All artifacts conform to JSON Schema | Every agent run (CI gate) |
| 2 | Per-file package match | Agent assigns files to the same packages as the answer key | Before each production release of the agent |
| 3 | Cycle detection correctness | Agent correctly identifies all circular dependencies in a known-cycles fixture | Before each production release |
| 4 | End-to-end build | The generated CMakeLists.txt + conanfile.py actually build successfully | After each package PR is merged |

---

## Level 1 — Schema Validation

**What:** Every JSON artifact produced by the agent is validated against its JSON Schema draft-07 definition in `specs/schemas/`.

**When:** Automatically on every agent run, before writing artifacts to the run directory. Also enforced in CI when plan PRs are opened.

**Pass criterion:** Zero schema validation errors for any artifact.

**Failure behavior:** Agent halts immediately and reports the failing field and schema path. Artifacts are not written to disk. The run status in `Manifest.json` is set to `error`.

**Implementation note:** Use the `ajv` library (or equivalent) for validation. Schema validation is not optional and cannot be bypassed.

---

## Level 2 — Per-File Package Match

**What:** Given a test fixture (a product/release with known build records), compare the agent's proposed `PackageAssignment.json` against the answer key (an already-migrated product with known correct assignments).

**Test fixture:**
> [OPEN] §8.1 — Which product/release is used as the Level 2 test fixture? See `open-questions.md §8.1`.

**Answer key:**
> [OPEN] §8.2 — Which already-migrated product is the Level 2 answer key? See `open-questions.md §8.2`.

**Metric:** Per-file assignment match rate. Agent assigns file F to package P; answer key assigns file F to package Q. Match = (P == Q).

**Pass criterion:**
> [OPEN] §8.3 — Minimum acceptable per-file match rate for Level 2. See `open-questions.md §8.3`.

**Known imperfections:**
> [OPEN] §8.4 — Are there known imperfections in the answer key (e.g., files that are assigned "wrong" in the migrated product but nobody fixed them)? See `open-questions.md §8.4`.

**How to run:**

```bash
node scripts/validate-assignment.js \
  --actual artifacts/runs/{runId}/PackageAssignment.json \
  --expected test-fixtures/answer-key/PackageAssignment.json \
  --output test-results/level2.json
```

**Output:** A report showing:
- Overall match rate
- Files that differ (with agent's proposed package vs. answer key package)
- Files unique to agent output (not in answer key)
- Files unique to answer key (not proposed by agent)

---

## Level 3 — Cycle Detection Correctness

**What:** Given a test fixture with known circular dependencies, verify the agent correctly identifies all cycles (no false negatives) and does not report false positive cycles.

**Test fixture:**
> [OPEN] §8.5 — A fixture product with known circular dependencies must be identified or constructed. See `open-questions.md §8.5`.

**Pass criterion:**

- **Recall:** Agent detects ≥ X% of known cycles (no false negatives below threshold).
- **Precision:** Agent reports ≤ Y% false positive cycles.

> [OPEN] §8.6 — Minimum recall and precision bars for Level 3 cycle detection. See `open-questions.md §8.6`.

**How to run:**

```bash
node scripts/validate-cycles.js \
  --actual artifacts/runs/{runId}/DependencyGraph.json \
  --expected test-fixtures/known-cycles/DependencyGraph.json \
  --output test-results/level3.json
```

---

## Level 4 — End-to-End Build

**What:** After the Implementation Agent generates `CMakeLists.txt` and `conanfile.py` for a package and the package PR passes CI, the package actually builds successfully with all supported macro combinations.

**When:** CI gate on every package PR. A package is considered "done" only when Level 4 passes.

**CI pipeline steps:**

1. **Schema validation** (Level 1) — validate all generated artifacts
2. **Conan install** — run `conan install . --profile <profile>` for each combination in `MacroMatrix.json`
3. **CMake configure** — run `cmake -B build -S .` for each combination
4. **CMake build** — run `cmake --build build` for each combination
5. **Include audit** — run `pr_reviewer` to check for unused/missing includes

**Pass criterion:** All steps pass for all macro combinations listed in `MacroMatrix.json` for the package.

**Failure handling:**

- CI failure triggers `compiler_error_tracer` (via `--fix-ci` flag)
- Error is logged to `BuildErrorReport.json`
- Fix is proposed and reviewed
- A follow-up PR is opened with the fix
- Level 4 must pass on the follow-up PR before the package is considered done

> [OPEN] §8.7 — Which CMake generator to use in CI (GNU make, Ninja, or Visual Studio)? See `open-questions.md §3.5`.
> [OPEN] §8.8 — What Conan profile(s) are used in CI? See `open-questions.md §8.8`.

---

## Test Fixture Requirements

To run Levels 2 and 3, the team needs:

1. **A product with complete scripted YAML build records** — the input for the agent
2. **An already-migrated version of the same product** — the answer key
3. **A documented list of known circular dependencies** in that product

These fixtures do not yet exist in this repository. They will be stored in:

```
test-fixtures/
├── answer-key/
│   └── PackageAssignment.json    # Manually curated correct assignments
├── known-cycles/
│   └── DependencyGraph.json      # Known SCCs for Level 3
└── build-records/
    └── {product}/{release}/      # Scripted YAML build records
```

> [OPEN] §8.1 — Test fixture selection is an open question. See `open-questions.md §1.1` and `§8.1`.

---

## Regression Testing

After the agent is deployed, regression tests ensure that updates to agent logic do not degrade assignment quality:

- Level 2 is run on every pull request to this repository that touches skill implementation code.
- Level 3 is run on every pull request that touches `fw_include_graph` or cycle detection logic.
- Regressions (drop in match rate or cycle recall) block merging.

---

## Acceptance Criteria Summary

| Level | Gate | Blocks |
|---|---|---|
| 1 | Zero schema errors | Every agent run |
| 2 | ≥ [TBD]% per-file match | Agent version release |
| 3 | ≥ [TBD]% cycle recall | Agent version release |
| 4 | All CI steps pass | Package PR merge |
