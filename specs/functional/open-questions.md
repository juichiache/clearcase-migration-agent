# Open Questions

All unresolved items from the design document, tagged by section. Each item has enough context for a team member to answer it without reading the full design doc.

**Status key:** `[OPEN]` = unanswered | `[ANSWERED]` = resolved | `[DEFERRED]` = explicitly deferred to v2

---

## §1 — Context and Scope

### §1.1 — Test Fixture Product/Release

**Question:** Which product and release should be used as the primary test fixture for Level 2 and Level 3 validation?

**Context:** The agent needs a product with complete scripted YAML build records AND an already-migrated version (the answer key). The fixture product should be representative of the overall codebase complexity but small enough to run tests quickly.

**Who can answer:** Migration lead + build system owner

**Needed by:** Before implementing Level 2/3 validation scripts

---

### §1.2 — Answer Key Product

**Question:** Which already-migrated product (if any) serves as the correctness answer key for Level 2 testing?

**Context:** We need a product where the file-to-package assignments are considered "correct" and stable. If the answer key has known imperfections, those should be documented (see §8.4).

**Who can answer:** Migration lead

**Needed by:** Before implementing Level 2 validation

---

### §1.3 — Known Circular Dependencies

**Question:** Are there known circular dependencies in the existing ClearCase codebase?

**Context:** If yes, they should be used as the Level 3 test fixture. If no known cycles exist, we need to either construct a synthetic fixture or defer Level 3 testing.

**Who can answer:** Senior architects familiar with the codebase

**Needed by:** Before implementing Level 3 validation

---

### §1.4 — Cross-Product Files

**Question:** Are there known files that appear in (are compiled by) multiple products?

**Context:** These "forkable" files need special handling in package assignment. If a file is compiled by both ProductA and ProductB, it may legitimately belong to a shared package, or it may be a copy. The agent needs a policy (see §5.1).

**Who can answer:** Build system owner / migration lead

**Needed by:** Before finalizing `existing_package_loader` and assignment logic

---

## §2 — Build Records

### §2.1 — YAML Schema from Python Extraction Script (CRITICAL)

**Question:** What is the exact YAML schema produced by the Python extraction script?

**Context:** The `build_record_ingester` skill cannot be implemented without knowing the exact YAML structure. Needed: a documented schema or at least a real example file showing all fields. Key fields expected: source file paths, per-file macro combinations, per-file included headers (per macro combo), library file sets.

**Who can answer:** Build system owner (owns the Python extraction script)

**Needed by:** Before implementing `build_record_ingester` — this is a blocker

**Urgency:** CRITICAL

---

### §2.2 — Products/Releases with YAML Today

**Question:** Which products and releases currently have scripted YAML build records? Which are in progress?

**Context:** The planning scope should be limited to products that have YAML records. Products without YAML cannot be processed by the agent.

**Who can answer:** Build system owner

**Needed by:** Before first planning run

---

### §2.3 — omake Makefile Parsing

**Question:** Should the agent parse omake makefiles as a fallback for products that don't have YAML build records yet?

**Context:** The design doc suggests probably not (given the YAML extraction script), but this needs team confirmation. omake parsing is significantly more complex than YAML ingestion.

**Who can answer:** Migration lead + build system owner

**Recommendation:** Defer to v2 unless a specific product urgently needs it before YAML is available.

---

## §3 — Planning Phase

### §3.1 — Copy-Map File Format (CRITICAL)

**Question:** What is the exact format of a copy-map file? Please provide a real example.

**Context:** The `copy_map_generator` skill cannot produce correct output without knowing the exact format. The `CopyMapProposal.schema.json` has a placeholder `entries[]` field. This must be updated with the real format once known.

**Who can answer:** Migration lead / team that owns the existing copy-map files

**Needed by:** Before implementing `copy_map_generator` — this is a blocker

**Urgency:** CRITICAL

---

### §3.2 — Agent-Generated Conan Patches

**Question:** Should the agent generate Conan patches for files that won't build clean, or should that be left entirely to human engineers?

**Context:** Conan patches are source-file modifications bundled with the Conan recipe. They are useful when a file needs minor changes (e.g., fixing include paths) to build in the new layout. The `patch_proposer` skill can generate these, but they add complexity and risk.

**Options:**
- (a) Agent proposes patches; human must approve before application
- (b) Agent flags the issue but does not propose a patch; human writes the patch manually
- (c) Conan patches are out of scope entirely

**Who can answer:** Migration lead + package owners

---

### §3.3 — Default Weight for Existing Package Assignments

**Question:** How strong should the default weight be for existing package assignments? What confidence bar must the agent clear before suggesting a reassignment?

**Context:** If a file is already assigned to package X in the copy-map, the agent should be conservative about suggesting it should move to package Y. But if build records strongly indicate Y is correct, the agent should surface that. What's the threshold?

**Options:**
- (a) Very conservative: only suggest reassignment if build records are unambiguous AND the current assignment causes a circular dependency
- (b) Moderate: suggest reassignment if build records clearly favor a different package
- (c) Aggressive: suggest reassignment whenever build records indicate a better fit

**Who can answer:** Migration lead

---

### §3.4 — Senior Engineer Breakdowns as Input

**Question:** Should the agent accept high-level package breakdowns from senior engineers as an additional prior input?

**Context:** Some senior engineers may have strong opinions about how files should be grouped (e.g., "all the networking files go in libnet, regardless of what the build records say"). If so, these breakdowns could be encoded as a higher-weight prior that overrides build-record evidence.

**Options:**
- (a) Yes — support a structured input format for senior engineer breakdowns
- (b) No — use only build records and existing copy-maps as priors

**Who can answer:** Migration lead + senior architects

---

### §3.5 — CMake Generator Target

**Question:** Which CMake generator should the agent target: GNU make, Ninja, or Visual Studio?

**Context:** The CMake generator determines the format of the generated `CMakeLists.txt` and the build command in CI. This affects which compilers and IDE integrations are supported.

**Who can answer:** Build infrastructure owner

**Needed by:** Before implementing `cmake_generator`

---

### §3.6 — Include Path Rewriting

**Question:** Is include path rewriting handled by the copy-map process, or is it a separate step?

**Context:** After migration, include paths in source files may need to change (e.g., from `#include "../../platform/include/foo.h"` to `#include "foo.h"`). Who handles this? The copy-map process, a separate script, or the agent?

**Who can answer:** Migration lead + package owners

---

## §4 — Implementation Phase

### §4.1 — Team-to-Folder Mapping

**Question:** Is there a mapping from team name to folder path (for routing PRs to the right team)?

**Context:** When the agent opens a package PR, it may need to set the correct reviewers or assign to the correct GitHub team. A team-to-folder or team-to-package mapping would enable this.

**Who can answer:** Migration lead

---

## §5 — File Assignment Edge Cases

### §5.1 — Forkable Files

**Question:** Which files (or file patterns) are considered "forkable" — allowed to appear in multiple packages?

**Context:** Forkable files can be assigned to multiple packages without triggering a duplicate assignment conflict. Examples might be common utility headers that are copied rather than referenced.

**Who can answer:** Migration lead + senior architects

---

### §5.2 — Orphan File Handling

**Question:** How should the agent handle files that appear in the ClearCase view but are absent from all build records (orphan files)?

**Options:**
- (a) Ignore them — don't include them in `PackageAssignment.json`
- (b) Flag them in a separate report but don't assign them
- (c) Attempt to assign them based on include graph proximity (which package most often includes them)

**Who can answer:** Migration lead

---

### §5.3 — Ambiguous Header Resolution

**Question:** When a header resolves to different files under different macro sets, what should the agent do?

**Example:** `#include "config.h"` resolves to `/vobs/product/config.h` under `MACRO_A=1` but to `/vobs/platform/config.h` under `MACRO_B=1`.

**Options:**
- (a) Pick the more common one, flag as conflict for human review
- (b) Refuse to make an assignment for that include edge; require human resolution
- (c) Model both resolutions in the include graph (edge splits into two with different macro contexts)

**Who can answer:** Migration lead + senior architects

**Recommendation:** Option (c) is most correct but most complex. Option (a) is pragmatic for the v1 agent.

---

### §5.4 — Circular Dependency Resolution

**Question:** What is the ranking algorithm for proposed cycle resolutions? Can the agent ever auto-resolve cycles, or does it always require human decision?

**Options:**
- Ranked resolution strategies (suggested order for v1):
  1. Extract interface header (lowest risk)
  2. Merge packages (medium risk — changes package count)
  3. Introduce forwarding/indirection (medium risk)
  4. Introduce a new shared package (high risk — affects many packages)
- Should the agent auto-resolve rank-1 resolutions without asking?

**Who can answer:** Migration lead + senior architects

---

### §5.5 — Dependency-Breaking Edit Conservatism

**Question:** How conservative should the `dependency_breaker` skill be?

**Options:**
- (a) Only obvious, low-risk edits (e.g., moving a forward declaration to a new header)
- (b) Broader edits allowed (e.g., splitting a header into two)
- (c) Report only — do not propose any code edits; leave dependency breaking entirely to humans

**Who can answer:** Migration lead + senior architects

---

## §6 — Design Decision Confirmations

### §6.1 — Build Records as Source of Truth

**Question:** Does the team confirm that scripted YAML build records are the source of truth for package assignment, not the folder tree?

**Context:** The agent's package assignment logic is built on this assumption. If the team prefers folder-based assignment as the primary signal, the agent's design needs to change significantly.

---

### §6.2 — One-Package-Per-PR with CI Gate

**Question:** Does the team confirm the one-package-per-PR with CI gate workflow for implementation?

**Context:** This is the key workflow constraint. It means the migration is sequential (one package at a time). If the team wants parallel package PRs (multiple packages simultaneously), the implementation agent design needs to change.

---

### §6.3 — Conan Options Derived Not Invented

**Question:** Does the team confirm that all Conan options must be derived from observed macro combinations?

**Context:** This rule prevents option proliferation. It means the agent cannot add options that aren't grounded in actual build record evidence. If some options are "conventional" and should always be present regardless of build record evidence, the team should document which ones.

---

### §6.4 — Package Assignment Fully Complete Before Building

**Question:** Does the team confirm that package assignment must be 100% complete (all files assigned and accepted) before any implementation PR is opened?

**Context:** Alternative: start implementing already-settled packages while planning continues for the rest. The current design requires full settlement first. If the team wants to pipeline planning and implementation, this needs architectural changes.

---

## §7 — Operating Environment

### §7.1 — Invocation Model

**Question:** Confirm VS Code + Remote-SSH + GitHub Copilot Chat as the sole invocation model.

**Context:** Are there other ways engineers will invoke the agent (e.g., a CLI tool, a web UI, a CI job)? The current design assumes VS Code + Copilot Chat.

---

### §7.2 — Workspace Repo

**Question:** What is the workspace repo where the Planning Agent emits plan PRs?

**Context:** The agent needs to know the org/repo to open PRs against. This should be a dedicated repository, not the source VOB repo.

**Needed by:** Before implementing plan PR workflow

---

### §7.3 — PR Reviewer Capabilities

**Question:** Confirm that both PR reviewer capabilities are needed: (a) unused/missing include audit and (b) duplicate package assignment audit.

**Context:** If (b) is not needed (because the plan PR already ensures no duplicates), the `pr_reviewer` scope can be reduced.

---

### §7.4 — Approval Authority

**Question:** Who has final approval authority for plan PRs? Is this per-team or centralized?

**Context:** The `Manifest.json` approval status is set when the plan PR is merged. Who is allowed to merge it? Is a single approver sufficient, or is consensus required?

---

## §8 — Validation and Testing

### §8.3 — Level 2 Pass Threshold

**Question:** What is the minimum acceptable per-file match rate for Level 2 testing?

**Context:** A 100% match rate is probably unrealistic (the agent may propose improvements over the answer key). But a floor is needed to block regressions.

**Suggestion:** 90% as a starting point, to be calibrated against the first real run.

---

### §8.4 — Answer Key Imperfections

**Question:** Are there known imperfections in the answer key (e.g., files assigned to the wrong package that were never fixed)?

**Context:** If yes, these should be excluded from the Level 2 comparison or flagged as "known divergences."

---

### §8.6 — Level 3 Cycle Detection Thresholds

**Question:** What are the minimum recall and precision thresholds for Level 3 cycle detection?

**Suggestion:** Recall ≥ 95% (we must not miss real cycles), Precision ≥ 80% (some false positives tolerable).

---

### §8.8 — Conan Profiles for CI

**Question:** What Conan profile(s) are used in the Level 4 CI build?

**Context:** The CI must build each package with each Conan profile that corresponds to an observed macro combination. The profiles must be defined and stored somewhere accessible to CI.

---

## §9 — Skills

### §9.1 — git_churn for Risk Scoring

**Question:** The `risk_scoring` skill ideally uses git churn (frequency of file changes) as a risk factor. But the source is in ClearCase, not Git. Options:

- (a) Drop churn from risk scoring entirely in v1 — score risk on complexity and include graph centrality only
- (b) Use ClearCase version history as a proxy for churn
- (c) Defer risk scoring to v2

**Who can answer:** Migration lead

**Recommendation:** Option (a) for v1 — simpler and still useful. Document the limitation.
