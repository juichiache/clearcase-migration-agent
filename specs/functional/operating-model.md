# Operating Model — Day-to-Day Usage

This document describes how engineers use the migration agent in practice: invocation, plan review, package implementation, and handling failures.

---

## Overview

The agent is invoked interactively from VS Code via GitHub Copilot Chat. The engineer SSH's into a machine with a ClearCase view mounted, opens this repository in VS Code, and chats with the agent.

**The full workflow:**

```
1. Engineer mounts ClearCase view on runner machine
2. Engineer connects via VS Code Remote-SSH
3. Engineer invokes Planning Agent → plan PR is opened
4. Team reviews and approves plan PR
5. Engineer invokes Implementation Agent for each package
6. Each package: PR opened → CI gate → merge → next package
```

---

## §7.1 Invocation

### Planning Phase

```
@agent plan --view-path /vobs/myproduct --release R2024.1
```

The agent will:
1. Confirm the view path is accessible
2. Report how many build records it found
3. Report how many files already have package assignments (loaded prior)
4. Ask clarifying questions for any `[OPEN]` items it cannot resolve automatically
5. Emit a summary at the end, with the plan PR URL

**Resuming a planning run:**

```
@agent plan --reuse-run plan-20241215T143022Z
```

Useful if the ClearCase view changes (new build records available) and you want to re-run planning without discarding prior Q&A decisions.

**Limiting scope:**

```
@agent plan --view-path /vobs/myproduct --release R2024.1 --product libcore
```

Runs planning for a single product within the release. Useful for incremental rollout.

### Implementation Phase

Start after the plan PR is approved and merged:

```
@agent implement --plan-approved --enable-writes --package libfoo
```

**Diagnose a CI failure:**

```
@agent implement --plan-approved --enable-writes --package libfoo --fix-ci
```

**Review a PR:**

```
@agent implement --review-pr 42
```

Note: `--review-pr` does not require `--enable-writes`. It is a read-only operation.

---

## §7.2 Plan Review

After the Planning Agent completes, it opens a PR to the workspace repo. The PR contains:

- All planning artifacts as committed files
- A summary comment with:
  - Total files, packages, and proposed changes
  - A table of `conflict` assignments requiring resolution
  - A list of detected circular dependencies with proposed resolutions
  - Open questions that need human answers before implementation

**Review checklist for approvers:**

- [ ] All `conflict` assignments have been resolved (or escalated to senior engineer)
- [ ] No circular dependencies remain unresolved
- [ ] Macro matrix looks reasonable (spot-check 2–3 packages)
- [ ] Locked packages were not touched
- [ ] `UserDecisions.json` captures any clarifications given during planning

> [OPEN] §7.2 — Workspace repo name/URL to be confirmed. See `open-questions.md §7.2`.
> [OPEN] §7.4 — Approval authority (who can approve plan PRs). See `open-questions.md §7.4`.

---

## §7.3 Package Implementation

After the plan is approved, implement one package at a time:

```bash
# Implement first package
@agent implement --plan-approved --enable-writes --package libfoo

# Wait for CI on the libfoo PR to pass, then:
@agent implement --plan-approved --enable-writes --package libbar
```

**The agent will not open the next PR** if the current package PR's CI has not passed. This is enforced by the `--ci-gate` check built into the implementation flow.

### If CI Fails

```
@agent implement --plan-approved --enable-writes --package libfoo --fix-ci
```

The agent reads the CI build log, classifies errors, and proposes fixes. Fixes are either:
- **Copy-map changes** — a file was assigned to the wrong package; fix by reassigning
- **Conan option changes** — a macro combination was missed; fix by adding an option
- **Code patches** — a source file needs a narrow change to break a circular include (always requires human approval)

The agent emits `BuildErrorReport.json` with its diagnosis and `PatchProposal.json` for any code patches.

---

## §7.4 Typical Day for a Migration Engineer

**Morning:**

1. Check open PRs in the workspace repo for any agent-proposed changes.
2. Review `conflict` assignments and resolve them by commenting on the PR.
3. Merge the plan PR if all issues are resolved.

**During the day:**

1. Invoke `@agent implement` for the next package in the queue.
2. Review the package PR opened by the agent.
3. Run `@agent implement --review-pr <number>` if the agent didn't auto-review.
4. If CI fails, run `@agent implement --fix-ci` and review the proposed fix.
5. Merge the package PR after CI passes.
6. Move to the next package.

**End of day:**

1. Check `ImplementationLog.json` to track overall progress.
2. Note any `[OPEN]` items that surfaced during the day in `open-questions.md`.

---

## §7.5 Escalation Paths

| Situation | Who to escalate to | Resolution |
|---|---|---|
| Circular dependency the agent cannot break | Senior architect | Manual resolution; record in `UserDecisions.json` |
| Conflict between agent proposal and team policy | Migration lead | Override decision; update `PackageAssignment.json` manually and re-run planning for affected packages |
| Build record YAML missing for a product | Build system owner | Provide YAML; re-run planning |
| Copy-map format unclear | Migration lead | Clarify and update `open-questions.md §3.1` |
| Locked package needs to be unlocked | Migration lead + package owner | Update lock flag and re-run planning |

---

## §7.6 Environment Requirements

| Requirement | Details |
|---|---|
| ClearCase view | Must be mounted on the runner machine before invocation |
| VS Code | With Remote-SSH and GitHub Copilot Chat extensions |
| GitHub Copilot | Active subscription; Chat enabled |
| Git | For plan PR and package PR operations |
| Node.js | Version 20+ (for the agent runtime) |
| Disk space | Sufficient for run artifacts (estimated: ~100 MB per planning run for large products) |

> [OPEN] §7.1 — Confirm VS Code + Remote-SSH + Copilot Chat as the sole invocation model. See `open-questions.md §7.1`.
