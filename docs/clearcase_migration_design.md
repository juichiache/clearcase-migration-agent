# ClearCase → CMake/Conan Migration: Agent Design

A working document for our session. By the end of the session, we'll have two artifacts:

1. **A filled-in version of this document** (your answers below)
2. **A finalized flow diagram** showing how the agent fits into your migration process

Those two artifacts — plus the Jurassic-refactor reference repo at https://github.com/tammym-demos/Jurassic-refactor — get handed to AI, which produces the implementation spec that GitHub Copilot will build from.

---

## What you told us

A summary of what we've taken from your emails about your current situation. **This is here as a reference and a sanity check — if anything is wrong, please correct it.** Otherwise, move on.

**Your environment and constraints**
- ClearCase has to be replaced, which also means moving off omake (it's ClearCase-licensed)
- Hundreds of GB of code in the VOB
- Omake makefiles list individual C/C++ source files and build large `-I` include sets using absolute paths rooted at the ClearCase virtual drive
- Heavy use of `#define` flags that vary by product — the same `#include` line resolves to different nested headers depending on the build
- Folder boundaries don't map cleanly to library boundaries because of how components reuse code across products

**Where you are in the migration**
- One product migrated; working on a second, more complex one
- Targeting CMake + Conan, with CMake generating any of GNU make / Ninja / Visual Studio projects
- Moving from "list every source file and header folder" to a standard "build libraries" model
- Generally one library, product, or header-only library per package (flexibility, not a hard rule)
- Libraries ideally owned by one development team
- Senior engineers have already produced high-level breakdowns of likely libraries; low-level packaging hasn't always been consistent with the high-level intent

**Your working rules**
- A source file generally lives in one package; exception is files that are OK to fork because divergence isn't a maintenance concern
- Build order is figured out iteratively — attempt to build, hit errors, trace through ClearCase build records to find where files were assigned
- Circular dependencies are resolved by some combination of: refactoring, splitting shared headers into a header-only library, or merging packages (with awareness that merging can make the package bigger and the dependencies worse)

**Your existing build flow**
- Partial GitHub repos containing CMake and Conan code
- A copy-map file in each repo, run during the build, that copies files out of ClearCase into the right location in the Git repo
- Conan patches applied to copied files when needed for the new build environment
- Build → test how well the copy-map worked → adjust copy-maps and repos → repeat
- You already have build records per product with separate logs per macro set; logs are big in total. You're generating new records for a large segment of the embedded products now, with more to follow.
- For each build you have two forms: the **raw** gcc command lines, and a **scripted** YAML output (run through your own Python script) that already extracts macros, the actual header files included per source file, all files associated with a library's build, and a few more things
- You already have many packages built (copy maps, conan files, CMake files), plus a separate folder of copy-maps annotated with intended package names that you use to work out dependencies before building
- You've found that some dependencies can only be broken by real code edits, which produces patch files and may require rebuilding to get updated dependencies. The codebase is very old with many flaws, and there are simple edits that would remove significant "false" dependencies.

**Where you've asked for AI's help**
- Making AI more effective as a PR reviewer
- Generating "more optimal" package assignments and catching files duplicated across packages
- Reviewing header includes and nested includes to remove unused ones (avoiding false dependencies)
- Ensuring source and header files include all needed headers, while minimizing public-header surface
- Generating CMake and Conan package files or assisting the engineers doing it
- Sifting build outputs to identify the macros each build uses — currently the most time-consuming manual step
- Offering simple code changes that break false dependencies
- Treating your current set of packages as a weighted input — accept them unless there's good reason to change — and growing the set incrementally, locking down packages as confidence grows

**What you've ruled out or pushed back on**
- A massive monorepo of ClearCase history in GitHub — expensive, multi-release, and you can't delete GitHub repos quickly if you need to redo
- Migrating any meaningful history before the new flow is proven to work
- Running Copilot tasks in GitHub Actions cloud when a runner with ClearCase view access would do the same job

**What you've said you want next**
- To proceed with the ideas, see how far they take us
- Working meetings to get you and Gerardo to the implementation-level understanding

> Corrections, if any: ____________

---

## What we're designing, in plain terms

We're designing an AI agent that plugs into the migration workflow you already have. Your workflow — partial GitHub repos with CMake and Conan code, a copy-map file that copies source out of ClearCase into the right location at build time, Conan patches for files that need adjustment, build-test-adjust iteration — stays exactly as it is. The agent works alongside you to do the most tedious parts faster, in two phases.

**Phase 1 — settle package assignment before building anything:**
- **Reading your scripted YAML build records** (macros, headers-included-per-source, library file sets) — the extraction work your Python script already does
- **Proposing which files belong in which package**, based on what gets co-compiled in your actual product builds, weighted by team ownership *and* by your existing set of packages (which it accepts unless there's good reason to change)
- **Working out dependencies** from copy-maps and the build records, and **detecting circular dependencies**, proposing how to resolve them — including simple code edits that break "false" dependencies
- Output: copy-maps with intended package names, iterated until the package picture is solid — with no building yet

**Phase 2 — generate and build, package by package:**
- **Generating draft CMakeLists.txt, conanfile.py, copy-map entries, and Conan patches** for each approved package
- **Reading compiler errors when a build fails** and proposing copy-map or option fixes — so the trial-and-error iteration loop you're in today shrinks
- **Reviewing PRs in GitHub** for unused or missing header includes and for files assigned to more than one package

As you grow confident in packages, they lock down and become higher-confidence inputs for the next round — an incremental ratchet rather than a one-shot migration.

The agent runs on a runner machine with a ClearCase view mounted. You and Gerardo interact with it through VS Code (connected to the runner via Remote-SSH) and GitHub Copilot. Source stays in ClearCase. No mass migration to GitHub.

The reference repo we're starting from (Jurassic-refactor) is a public demo that does something similar for general legacy refactoring. We're going to adapt it to your specific situation by answering the questions in this document.

---

## How AI helps you during this session

You don't need to know the Jurassic-refactor repo to fill in this doc. Open it in VS Code with GitHub Copilot Chat enabled, and use Copilot in three specific ways as we go:

- **When a term comes up:** "Explain what skill X in the Jurassic-refactor repo does in plain language." Copilot reads the repo and tells you.
- **After you answer a section:** "Does this answer conflict with anything else in the doc?" Copilot checks the doc for internal consistency.
- **At the very end:** "Synthesize the final flow diagram based on this doc." Copilot reads the filled-in doc and produces the Section 11 diagram for us to refine together.

We'll do all three live in the session.

---

## 1. Your sample for this project

We need one concrete slice of your code as the test bed — the legacy sample the agent will be built against. **This answer drives every other technical decision in this doc**, because it determines what the agent has to handle in v1.

**Which product and release should we use?**

- Product: ____________
- Release: ____________
- Approximate file count: ____________

> *Why we're asking:* this becomes the test fixture the AI builds the agent against. Smaller is better for speed; representative is better for honesty.

**Are there known problem areas in this sample we should make sure the agent handles?**

- Known circular dependencies between would-be packages: ____________
- Files in this sample that also appear in other products with different macro combinations: ____________

> *Why we're asking:* these are the exact failure modes that make naive packaging tools fail. We need the agent designed to handle them from day one.

**Already-migrated product as the answer key.**

You've migrated one product fully and are working on a second. The migrated product is our correctness test bed — we cover how we'll use it in detail in Section 8. For now, just flag it:

- Which product(s) are already done and could serve as the "known-good" answer key? ____________

> *Why we're asking:* the migrated product is the only place we have both the inputs and a verified output, which makes it the strongest test signal. Section 8 lays out exactly how we use it. The legacy sample above (the complex in-progress slice) is the separate "stress" test; the migrated product is the "correctness" test.

---

## 2. Your build records

You have build records in two forms for each build: the **raw** gcc command lines, and a **scripted** YAML output where your own Python script has already extracted the macros, the actual headers included per source file, the files associated with each library's build, and more. You're also actively generating new records across a large segment of the embedded products, with more to come.

**Which form should we feed the agent — raw, scripted YAML, or both?**

Our recommendation: **scripted YAML as the primary input, raw as a fallback.** Your script has already done the extraction work the agent would otherwise have to do, and you've validated that output. The agent consumes the YAML directly. The raw gcc command lines stay available for the cases where the agent needs something the script didn't capture (or to spot-check the YAML). This means the agent's `build_record_ingester` is mostly a YAML schema reader, not a from-scratch log parser — which makes it smaller, faster to build, and more reliable.

> Confirm this approach, or tell us if you'd rather the agent work from raw output: ____________

**Can you share the YAML schema (or a sample) your script produces?**

> ____________

> *Why we're asking:* the agent's most important input is your YAML. The implementation spec needs the schema so the agent reads every field your script emits — macros, per-source headers, library file sets, and the "few more things." If the schema is still evolving as you generate new records, tell us what's stable vs. in flux.

**Coverage.** Which products and releases do you have scripted YAML for today, and which are still being generated?

> ____________

**One specific question:** do you also want the agent to parse your omake makefiles themselves (which encode the *intent* — how `-I` lists are built, what conditional includes mean), or is the build-record data sufficient? Given your script already extracts actual headers-included-per-source, the makefiles may be redundant — but you'll know whether there's intent in them that the build records lose.

> ____________

> *What it changes in the design:* whether we build an `omake_parser` capability for v1 or rely entirely on your YAML.

---

## 3. Your existing migration artifacts

You already have a working flow for partially-migrated packages. The agent has to fit *into* that flow, not replace it. We need to understand the shape of your existing artifacts so the agent reads and writes them correctly.

**3.1 Copy-map file format.** Can you share an example copy-map from a current partial repo? We need to know its format so the agent can both read existing copy-maps and emit new ones in the same shape.

> Format / example file: ____________

> *Why we're asking:* the Implementation Agent's primary output is a copy-map per package. It has to match the format your existing build process expects.

**3.2 Conan patches.** Some of your packages apply patches to files copied from ClearCase. Should the agent generate proposed patches when it identifies a file that won't build clean as-is, or should patches stay a fully human-authored step?

> Your answer: ____________

> *What it changes in the design:* if the agent generates patches, the Implementation Agent gets an additional output type (`patches/`) and the compiler-error-tracer learns to propose patch hunks. If not, the agent only proposes copy-map and CMake/Conan changes, and humans handle patches.

**3.3 Your existing packages as a weighted starting point.** This is central to how the agent should work. You already have many packages built (copy maps, conan, CMake), plus a folder of copy-maps annotated with intended package names. You've told us: treat the current set of packages as an input with reasonable weight — the agent should accept them unless there's a good reason to change — and grow the set incrementally from there, locking down packages as confidence in them increases.

So the agent's behavior is:
- Load your existing packages as a weighted prior, not a blank slate
- Propose new packages and adjustments for the unpackaged code
- Only suggest changing an existing package when there's a clear reason (e.g., it would resolve a circular dependency or a duplicate assignment), and surface that reason for your review
- As you approve packages, they become locked and feed back in as higher-confidence priors for the next iteration

> Confirm this matches your intent, and tell us how strong the default weight should be — i.e., how high a bar should the agent clear before suggesting a change to an already-built package? ____________

> *What it changes in the design:* the Planning Agent takes your existing packages + the annotated copy-map folder as a primary input, and its output is a *diff* against the current state (accept / add / suggest-change-with-reason), not a from-scratch partition. There's also a notion of "locked" packages that the agent won't touch.

**3.4 Senior-engineer high-level breakdowns.** Separate from the built packages, your senior engineers have produced high-level library breakdowns. Should the agent use these too, as a higher-level guide alongside the existing packages?

> Your answer: ____________

**3.5 Build system generator target.** CMake can produce GNU make, Ninja, or Visual Studio project files. Are you targeting one of these specifically, or do you need the agent to support all three?

> Your answer: ____________

> *What it changes in the design:* generally a configuration concern (CMake handles it), but the agent's CMakeLists output needs to be tested against whichever generators you're using.

**3.6 Include path rewriting.** Your current code uses absolute `-I` paths rooted at the ClearCase virtual drive. Is rewriting these to repo-relative paths handled by your existing copy-map process, or is that a separate step the agent should propose changes for?

> Your answer: ____________

> *What it changes in the design:* if the copy-map handles it, the agent just maintains the convention. If not, the agent needs a "rewrite includes" capability as a generation step.

---

## 4. Your team structure

The agent will propose package boundaries. One important factor is that packages should ideally be owned by a single team — you said this explicitly.

**Team-to-folder mapping.** We need a rough list of which team owns which folders. A draft is fine.

- Who can produce this, and by when? ____________

> *Why we're asking:* without this, the agent will propose boundaries based purely on code coupling, which will sometimes cut across team lines.
>
> *What it changes in the design:* if you provide this, the agent treats team ownership as a constraint when proposing boundaries. If you don't, it uses coupling only and you'll have to re-shuffle proposals manually for ownership.

---

## 5. Specific cases the agent will hit

These are the hard cases. Each one needs your policy decision so the agent knows what to *do*, not just detect.

**5.1 A file is allowed to be in multiple packages.** You mentioned some files are OK to fork across packages because divergence isn't a maintenance concern. What patterns or examples should the agent treat as "forkable"?

> ____________

> *Why we're asking:* default behavior treats any file appearing in two packages as a bug to flag. Your answer turns off that flag for specific cases.

**5.2 A file appears in zero products' build logs (orphan).** What should the agent do?
- (a) Ignore it
- (b) Flag for human review
- (c) Try to assign it to a package based on which files include it

> Your choice: ____________

> *What it changes in the design:* this becomes a code branch in the Planning Agent's package assignment logic.

**5.3 Two products' build logs say `foo.h` resolves to different files under the same macros.** What should the agent do?
- (a) Pick one and flag the conflict
- (b) Refuse to proceed until a human resolves it
- (c) Model the file as belonging to both contexts (and warn that this may produce duplicate Conan options)

> Your choice: ____________

> *What it changes in the design:* affects how the agent represents include resolution in its internal graph.

**5.4 Circular dependency with no clean break.** Our proposed ranking of resolutions (please confirm or reorder):
1. Simple dependency-breaking code edit, where one exists (you noted the codebase has many simple changes that remove significant false dependencies — see 5.5)
2. Extract shared headers into a header-only library (medium work, often the right answer)
3. Larger refactor to break the cycle (most work, cleanest result)
4. Merge the packages (least work, but creates bigger packages with more total dependencies)

> Your ranking, and whether the agent should pick automatically or always ask a human: ____________

> *What it changes in the design:* directly encodes into the Planning Agent's cycle-resolution logic.

**5.5 Simple code edits to break false dependencies.** You've told us the codebase is old, has many flaws, and contains simple edits that would remove significant "false" dependencies — and that breaking these sometimes requires real code changes captured as patch files, possibly with a rebuild to get the updated dependencies. Should the agent propose simple dependency-breaking code edits (as patches), and if so, how conservative should it be?

- (a) Propose only the most obvious, low-risk edits (e.g., removing an unused include that creates a false dependency), flag for human approval
- (b) Propose a broader set of edits including small code restructuring, flag for human approval
- (c) Only identify and report false dependencies; never propose the edit itself

> Your choice, and any kinds of edits that should always/never be attempted: ____________

> *What it changes in the design:* this brings back a constrained `code_refactor` capability (see Section 9) focused narrowly on dependency-breaking, distinct from general refactoring which stays out of scope. It also means the iteration loop includes an optional "edit → patch → rebuild → new dependencies" cycle.

---

## 6. How the agents make decisions

Separate from the question of *what* the agents do (skills) and *what you do with them* (operating model), there are a few principles that govern *how* the Planning and Implementation agents behave. These are the guardrails that prevent the agents from doing the wrong thing efficiently. Please confirm or push back on each.

**6.1 Build records are the source of truth, not the folder tree.** In a normal codebase the folder layout roughly predicts package boundaries. In yours it doesn't — products pull arbitrary file sets from arbitrary folders with arbitrary macro combinations. The Planning Agent will infer package boundaries from what files are co-compiled in your build records, weighted by macro context — *not* from where files happen to sit in the ClearCase tree.

> Confirm, or push back: ____________

> *What it changes in the design:* this is the foundational principle behind the entire build_record_ingester approach. If you'd rather the agent start from the folder tree and use build records as confirmation, that's a different design.

**6.2 One package per PR, with a verification gate.** The Implementation Agent will produce one PR per package, and won't submit a PR for package N+1 until package N's CI is green. This keeps reviews bounded and ensures failures are localized.

> Confirm, or propose alternative: ____________

> *What it changes in the design:* this is a workflow guardrail in the Implementation Agent. If you'd prefer a different cadence (multiple packages per PR, or no gating), the agent's sequencing logic changes.

**6.3 Conan options must be derived, not invented.** When the Implementation Agent writes a `conanfile.py`, the options block must enumerate exactly the macro combinations observed in your build records for that package's source files. No options that aren't observed; no collapsing options that vary across products. This is the difference between a `conanfile.py` you can maintain and one that hallucinates flexibility you don't need.

> Confirm: ____________

> *What it changes in the design:* this is a hard constraint on the `macro_matrix` skill's output. Without it, you could get a Conan options matrix that looks reasonable but doesn't match how your code is actually built.

**6.4 Package assignment comes fully before any building.** You confirmed you want to settle copy-map / package assignment first — working out dependencies from the copy-maps and build records (especially headers-included-per-source) — and iterate on that until the package picture is solid, *before* generating any Conan/CMake or building anything. So the agent has a distinct first phase that produces and refines package assignments and a dependency graph, with no build artifacts, matching the annotated copy-map folder you already maintain. Conan/CMake generation and building only begin once you're happy with the assignment.

> Confirm, or tell us if you'd rather interleave assignment and building: ____________

> *What it changes in the design:* this splits the Planning Agent into two clear stages — (1) package assignment + dependency resolution, output as copy-maps with intended package names, and (2) Conan/CMake generation — with a human checkpoint between them. The first stage is cheap to iterate because nothing is built.

---

## 7. How you'll actually use the agent

Once built, you and Gerardo work with the agent day-to-day. We need to design for how you'll actually run it.

**7.1 Invocation surface.**

Proposed: VS Code on your laptop, connected via Remote-SSH to the runner with the ClearCase view, agent invoked through GitHub Copilot Chat.

> Confirm, or propose alternative: ____________

**7.2 Plan review.**

Proposed: the Planning Agent emits its proposed plan as a pull request against a small workspace repo on GitHub; you and Gerardo review and approve the PR before the Implementation Agent runs.

> Confirm, or propose alternative (standalone file? in-VS-Code UI?): ____________

**7.3 AI as PR reviewer.** You asked specifically about making AI more effective in your existing PR review process. Two concrete capabilities we'd add:

- Reviewing PRs for unused header includes and for nested includes that could be removed without breaking builds (avoiding false package dependencies)
- Reviewing PRs to ensure source and header files include all needed headers, while minimizing what gets included in public-facing headers

> Confirm both are wanted, or specify which: ____________

> *What it changes in the design:* this adds a separate `pr_reviewer` capability that runs on incoming PRs in GitHub, in addition to the Planning + Implementation agents that run on the runner.

**7.4 Approval authority.** Who has the authority to approve a Planning Agent plan and let the Implementation Agent proceed?

> ____________

---

## 8. How we'll know it's working

Your already-migrated product is the single most valuable thing you have for testing the agent, because it's the only place where you have both the *inputs* (the ClearCase source and build records as they were before migration) and a *known-correct output* (the finished packages, copy-maps, conanfiles, CMake, patches, and build order your engineers produced). That pairing lets us run a **holdout test**: hide the human's answer, run the agent on the same inputs, and compare. Every match builds confidence; every mismatch is diagnostic — it points exactly at where the agent's logic diverges from your engineers' judgment.

We propose four levels of test, each using the migrated product as the answer key.

**Level 1 — Structural.** The agent's outputs have valid shape (well-formed copy-maps, parseable CMakeLists, valid conanfile.py syntax). AI handles this automatically with no input needed from you.

**Level 2 — Package assignment (the most important).** Give the agent the build records and ClearCase view for the migrated product, but withhold the human's final package assignment. The agent proposes its own boundaries and copy-maps. We then measure per-file agreement: of all files the human assigned, what fraction did the agent place in the matching package?

- What's the minimum acceptable per-file match? X = ____________

The disagreements are where the real value is. We sit with your senior engineers and triage each one into: (a) agent was wrong, (b) agent was acceptably different, or (c) agent was right and the original human assignment was suboptimal. That triage is itself a design exercise — it surfaces the implicit rules your engineers used that we then turn into explicit agent instructions.

> *Why we're not aiming for 100%:* you've told us low-level packaging hasn't always matched high-level intent, so the answer key itself has imperfections. Aiming for a literal 100% match would train the agent to reproduce those. The goal is high agreement plus a clean explanation for every disagreement.

**Level 3 — Dependency and cycle detection.** Separately from where files land, check whether the agent reconstructs the same dependency graph and finds the same circular dependencies your engineers had to resolve in the migrated product. You know which cycles required header extraction, which required code edits, which were merged.

- Bar: agent finds all the cycles the humans actually hit (a missed cycle is a serious bug); agent-flagged cycles the humans didn't hit get investigated as possible false dependencies. Confirm or adjust: ____________

**Level 4 — Generation and end-to-end build.** Two sub-tests:

- *Generation:* feed the agent the human's fixed package boundaries and check whether its generated conanfile.py options match the human's. Since the human's conanfile already enumerates the right macro combinations, this is a direct test of the `macro_matrix` logic.
- *End-to-end:* run the agent's complete output for a package through your actual copy-map → Conan/CMake pipeline and confirm it builds.

- What's the end-to-end bar? ____________
  - Package builds in isolation against its declared dependencies, or
  - Full product builds end-to-end through the existing pipeline, or
  - Something in between

> *Why this is separate from day-to-day iteration:* a full build is the most honest signal but the most expensive to run. The Level 2 per-file comparison is cheap, so it's what we iterate against while tuning the agent; Level 4 is the periodic confirmation that the output actually works.

**Two fixtures, two jobs.** The migrated product is a *correctness* fixture — it proves the agent reproduces known-good results. But it's also your simpler product, so it doesn't prove the agent handles the hard cases (heavy macro variation, deep cycles, cross-product files) that your in-progress complex product has. We pair it with a slice of the complex product as a *stress* fixture (this is the legacy sample in Section 1). Strong performance on the migrated product is necessary but not sufficient; the stress fixture is where we confirm the agent handles what actually makes your migration hard.

- Which migrated product (and which packages within it) make the best answer key? ____________
- Anything about that migration we should know — known imperfections, packages your engineers would do differently in hindsight? ____________

---

## 9. What the agent will do internally (skills)

This is the most technical section, but it's also the one where Copilot can help you most. For each row below, you can ask Copilot Chat to explain the skill in plain language as we go through it.

Most of these are pre-decided based on our conversation — you only need to confirm. A few rows have specific questions where we need your input (marked **Question for you**).

| Skill | What we're proposing | Your call |
|---|---|---|
| `repo_snapshot` | **Modify** to read from a ClearCase view path, not a Git working directory. | |
| `fw_include_graph` | **Modify** so dependency edges carry macro context: an edge is `(includer, includee, macro_set)`. Note: your scripted YAML already records actual headers included per source, so this skill consumes that rather than re-deriving it. | |
| `git_churn` | **Question for you:** churn data helps flag risky files for extra review. You don't have Git history, but ClearCase has `cleartool lshistory`. Options: (a) drop churn entirely, (b) extract from ClearCase for hot files only, (c) defer this for v2. | |
| `risk_scoring` | **Modify** to work with whatever feeds it (depends on `git_churn` answer above). | |
| `dependency_breaker` (was `code_refactor`) | **Bring back in constrained form (per Section 5.5).** Proposes simple, low-risk code edits that break false dependencies, emitted as patches for human approval. NOT general refactoring — narrowly scoped to dependency removal. | |
| `dependency_upgrader` | **Drop.** Not relevant — you're packaging existing code, not upgrading library versions. | |
| `pr_writer` | **Modify** to write to your existing partial per-package repos via PR. | |
| `pr_reviewer` (new) | **New, depends on Section 7.3.** Reviews PRs in GitHub for unused/missing header includes and duplicate package assignments. | |
| `omake_parser` (new) | Covered in Section 2. Likely unnecessary given your scripted YAML already extracts intent — your answer there decides. | |
| `build_record_ingester` (new) | **New, in v1 — but smaller than before.** Reads your scripted YAML (macros, per-source headers, library file sets) directly; falls back to raw gcc command lines only for anything the YAML doesn't cover. Mostly a schema reader, not a from-scratch log parser. | |
| `existing_package_loader` (new) | **New, in v1 (per Section 3.3).** Loads your current packages and annotated copy-maps as a weighted prior; tracks which are "locked." The Planning Agent's output is a diff against this, not a from-scratch partition. | |
| `macro_matrix` (new) | **New, in v1.** Takes per-file macro contexts (from the YAML) and produces the minimal set of Conan options. The biggest time-save vs. your current manual macro work. | |
| `header_pruner` (new) | **New, in v1 if Section 7.3 confirmed.** Identifies unused includes in source and header files; flags missing includes being inherited transitively. | |
| `copy_map_generator` (new) | **New, in v1.** Reads the Planning Agent's package boundaries and emits a copy-map in your existing format (per Section 3.1). This is the output of the assignment-first phase (Section 6.4). | |
| `patch_proposer` (new) | **New, depends on Section 3.2.** When a file won't build clean as-copied, proposes a Conan patch. Shares machinery with `dependency_breaker`. | |
| `compiler_error_tracer` (new) | **New, in v1.** Reads failed build errors, traces to ClearCase view, proposes copy-map or option fixes. The iteration-loop speedup. | |
| Anything we're missing? | | |

> *Why this matters:* each row here is a piece of the agent's capability. The set of skills determines what boxes appear in the final flow diagram.

> Your additions, overrides, or pushback: ____________

---

## 10. What we're explicitly not doing

Calling these out so scope doesn't drift while the agent is being built. We'll review and refine this list together.

- Migrating the ClearCase VOB to a GitHub monorepo
- Replaying full ClearCase history into Git
- General source code refactoring — *except* the narrow, simple dependency-breaking edits in Section 5.5, which are in scope
- Header reorganization in phase one (header pruning per Section 7.3 is in scope, but reorganizing where headers live is not)
- `#define` cleanup in phase one
- Conan version upgrades
- Running agents in GitHub Actions cloud (we're using your runner instead)
- Anything that requires deleting GitHub repos (because of the multi-day IT process)
- Replacing omake with the agent — the agent reads existing build records, doesn't substitute for the build

> Additions: ____________

> Strikes: ____________

> *Why this matters:* the AI building the agent reads this list. Anything on it gets explicitly excluded from the implementation, which keeps the build focused and prevents Copilot from drifting into adjacent work.

---

## 11. Final flow diagram

*Produced at the end of the session, after the rest of this doc is filled in.*

We'll ask Copilot to synthesize a first draft based on everything above, then refine it together on the whiteboard. The diagram shows:

- The agent's inputs: your scripted YAML build records (Section 2), your existing packages + annotated copy-maps as a weighted prior (Section 3.3), team mapping (Section 4), the legacy sample (Section 1)
- **Phase 1 — package assignment** (Section 6.4): the agent works out package boundaries and dependencies from copy-maps + YAML, output as copy-maps with intended package names, iterated until solid, *no building*
- Human checkpoint on the assignment
- **Phase 2 — generation + build**: Conan/CMake generation, then the existing copy-map → build → adjust loop, with optional dependency-breaking edits (Section 5.5)
- The agent's internal skills (Section 9) and decision principles (Section 6)
- The human approval gates (Section 7) and PR review loop (Section 7.3)
- The validation gates (Section 8) and exception-handling branches (Section 5)
- The incremental ratchet: approved packages become "locked" priors feeding the next iteration

*[Diagram goes here]*

---

## What happens after this session

We hand the filled-in doc plus the Jurassic-refactor repo to AI. AI produces:

- The implementation spec (a detailed engineering doc)
- The build prompt and sequencing for GitHub Copilot to actually construct the agent
- The validation plan for testing the build against your legacy sample
- The runtime operating doc — how to actually use the agent once it's working

Anything in this doc still blank or vague gets flagged by AI before the build kicks off, and we resolve it then.
