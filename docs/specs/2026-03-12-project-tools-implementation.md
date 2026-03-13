# claude-code-commands Plugin Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert `/map` and `/adversarial-review` into an installable Claude Code plugin with fully polyglot-adaptive discovery patterns.

**Architecture:** Two markdown command files packaged as a Claude Code plugin. Specs are the source of truth; command files are regenerated from specs. All static file/pattern lists use a three-part detect/examples/adapt structure.

**Tech Stack:** Claude Code plugin format (`.claude-plugin/plugin.json`), markdown command files, git

**Spec:** `docs/specs/2026-03-12-claude-code-commands-plugin-design.md`

---

## File Structure

| File | Action | Purpose |
|------|--------|---------|
| `.claude-plugin/plugin.json` | Create | Plugin metadata |
| `.gitignore` | Create | Git ignore rules |
| `LICENSE` | Create | MIT license |
| `README.md` | Create | Plugin description and install instructions |
| `docs/specs/SPEC-map.md` | Move + Edit | Source of truth for /map (move from repo root) |
| `docs/specs/SPEC-adversarial-review.md` | Move + Edit | Source of truth for /adversarial-review (move from repo root) |
| `commands/map.md` | Create | Executable /map command (generated from spec) |
| `commands/adversarial-review.md` | Create | Executable /adversarial-review command (generated from spec) |

---

## Chunk 1: Scaffold and Reorganize

### Task 1: Create plugin structure

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `.gitignore`
- Create: `LICENSE`
- Create: `README.md`
- Create: `commands/` (directory)

- [ ] **Step 1: Create `.claude-plugin/plugin.json`**

```json
{
  "name": "claude-code-commands",
  "description": "Project mapping and adversarial review commands for Claude Code — polyglot-adaptive, works on any codebase",
  "version": "1.0.0",
  "author": {
    "name": "Matthew Belchak"
  },
  "license": "MIT",
  "keywords": ["architecture", "review", "mapping", "polyglot", "claude-code-commands"]
}
```

- [ ] **Step 2: Create `.gitignore`**

```
.DS_Store
*.swp
*~
```

- [ ] **Step 3: Create `LICENSE`**

Standard MIT license with copyright `Matthew Belchak`.

- [ ] **Step 4: Create `README.md`**

```markdown
# claude-code-commands

Polyglot-adaptive project mapping and adversarial review commands for Claude Code.

## Commands

- `/map` — Generate or update `ARCHITECTURE.md` for any project
- `/adversarial-review` — Find gaps, contradictions, and missing pieces in project docs and config

## Install

```bash
claude plugins install /path/to/this/repo
```

## Design

See `docs/specs/` for detailed specifications.
```

- [ ] **Step 5: Create empty `commands/` directory**

Run: `mkdir -p commands`

- [ ] **Step 6: Commit scaffold**

```bash
git add .claude-plugin/ .gitignore LICENSE README.md commands/
git commit -m "chore: scaffold plugin structure"
```

---

### Task 2: Move specs to docs/specs/

**Files:**
- Move: `SPEC-map.md` → `docs/specs/SPEC-map.md`
- Move: `SPEC-adversarial-review.md` → `docs/specs/SPEC-adversarial-review.md`

- [ ] **Step 1: Move spec files**

```bash
git mv SPEC-map.md docs/specs/SPEC-map.md
git mv SPEC-adversarial-review.md docs/specs/SPEC-adversarial-review.md
```

- [ ] **Step 2: Commit move**

```bash
git commit -m "chore: move specs to docs/specs/"
```

---

## Chunk 2: Update SPEC-map.md

### Task 3: Apply Group 1 fixes to SPEC-map.md (adaptive discovery)

**Files:**
- Modify: `docs/specs/SPEC-map.md`

Reference: Design spec "The Hybrid Adaptive Pattern" → "Applied Areas" section for exact example sets.

- [ ] **Step 1: Update structure-mapper agent prompt — manifest discovery**

In the structure-mapper agent prompt (around line 291), replace:
```
1. Check for manifest files: package.json, Cargo.toml, pyproject.toml, go.mod, Gemfile, pom.xml, etc.
```
With:
```
1. Check for manifest files (examples, not exhaustive): package.json, Cargo.toml, pyproject.toml, go.mod, mix.exs, build.sbt. Also check the project root for any other build, package, or dependency files specific to the detected language.
```

Also update the embedded command file section (around line 530) with the same change.

- [ ] **Step 2: Update structure-mapper — source directory discovery**

Replace source directory instruction (around line 292) to include full adaptive pattern:
```
2. First run `ls` on the project root to discover the actual directory layout. Do NOT limit yourself to common patterns — explore ALL directories that contain source code. Common examples include `src/`, `lib/`, `app/`, `pkg/`, `cmd/`, `internal/`, `Sources/`, `crates/`, `packages/`, but always adapt to what actually exists. Do not assume `src/` is the only source directory. If no language can be determined from manifests or file extensions, treat all non-binary files as potential source code. Note the uncertainty in the output.
```

Also update the embedded command file section with the same change.

- [ ] **Step 3: Update structure-mapper — test file globs**

Replace test file globs (around line 293). Change ALL `*.*` patterns to `*`:
```
3. Glob for test files in common directories: tests/**/*, test/**/*, spec/**/*, __tests__/**/* Also check for language-specific conventions (examples, not exhaustive): *_test.go (Go), test_*.py (Python), *_spec.rb (Ruby), t/*.t (Perl), src/test/**/* (Java/Maven). In Rust, look for inline #[cfg(test)] modules. For any other language detected, check for its test file naming conventions.
```

Also update the embedded command file section with the same change.

- [ ] **Step 4: Update structure-mapper — config file globs**

Replace config file globs (around line 294):
```
4. Glob for config files (examples, not exhaustive): .eslintrc.* (JS), rustfmt.toml (Rust), .rubocop.yml (Ruby), mypy.ini (Python), .golangci.yml (Go), .formatter.exs (Elixir). Also search the root for any other dotfiles and YAML/TOML/JSON files that appear to be linter, formatter, or build tool configuration for the detected language.
```

Also update the embedded command file section with the same change.

- [ ] **Step 5: Update dependency-mapper — import patterns**

This is the most critical polyglot fix. Replace the dependency-mapper's step 1 (around line 311):
```
1. First identify the project's language(s) from manifests and file extensions. Then grep for language-appropriate import patterns (examples, not exhaustive): `import`/`from` (Python/JS/TS/Java/Go/Swift/Dart), `require`/`require_relative` (Ruby/Node), `use`/`mod` (Rust), `using` (C#), `#include` (C/C++), `open` (OCaml). For any language not listed, research its module/import system and grep for the appropriate keywords. Only search source files, not comments or documentation. If no language can be determined, search broadly for import-like statements.
```

Also update the embedded command file section with the same change.

- [ ] **Step 6: Update convention-scanner — linter configs**

Replace linter config list (around line 349):
```
2. Read any linter/formatter configs (examples, not exhaustive): .eslintrc, .prettierrc (JS), rustfmt.toml (Rust), .rubocop.yml (Ruby), .pylintrc, .flake8 (Python), .golangci.yml (Go), .clang-format (C/C++), .editorconfig. Also search root for any other linter/formatter config files specific to the detected language.
```

Also update the embedded command file section with the same change.

- [ ] **Step 7: Update convention-scanner — language configs**

Replace language config list (around line 350):
```
3. Read the language-specific config if present (examples, not exhaustive): tsconfig.json (JS/TS), pyproject.toml (Python), Cargo.toml (Rust), go.mod (Go), Gemfile (Ruby), pom.xml/build.gradle (Java), mix.exs (Elixir), build.sbt (Scala). Check for any other language-specific project configuration files.
```

Also update the embedded command file section with the same change.

- [ ] **Step 8: Update convention-scanner — CI config**

Replace CI config list (around line 352):
```
5. Read CI config if present: .github/workflows/**/*, .gitlab-ci.yml, .circleci/**/*, Jenkinsfile*, .buildkite/**/*, .travis.yml, azure-pipelines.yml, bitbucket-pipelines.yml. Check for any other CI/automation configuration present.
```

Also update the embedded command file section with the same change.

- [ ] **Step 9: Update convention-scanner — AI assistant docs**

Replace convention docs check (around line 353):
```
6. Check for CLAUDE.md, AGENTS.md, GEMINI.md, COPILOT.md, CURSOR.md, AIDER.md, .cursorrules, .windsurfrules, .clinerules, .github/copilot-instructions.md, CONTRIBUTING.md, or any root-level markdown or dotfile that appears to contain AI assistant instructions or contribution guidelines.
```

Also update the embedded command file section with the same change.

- [ ] **Step 10: Search for and replace ALL remaining `etc.` catch-alls**

Search SPEC-map.md for every instance of `, etc.`, `etc.)`, `or similar`, `or equivalent`. Replace each with the appropriate adaptive instruction from the hybrid pattern. Verify zero remain.

- [ ] **Step 11: Commit Group 1 fixes for SPEC-map.md**

```bash
git add docs/specs/SPEC-map.md
git commit -m "spec(map): apply hybrid adaptive pattern to all discovery lists"
```

---

### Task 4: Apply Groups 2–4 fixes to SPEC-map.md

**Files:**
- Modify: `docs/specs/SPEC-map.md`

- [ ] **Step 1: Fix #1 — Remove all `*.*` glob patterns**

Search for every `*.*` in the file. Replace with `*`. This affects test globs in the structure-mapper agent prompt AND the embedded command file. Verify zero `*.*` patterns remain.

- [ ] **Step 2: Fix #2 — Verify section heading mapping table exists**

The argument-to-section mapping table already exists (lines 38-41). Verify it's correct:
- `structure` → `## Structure`
- `deps` → `## Dependencies`
- `conventions` → `## Conventions`
- `impact` → `## Change Impact Map`

- [ ] **Step 3: Fix #15 — Update tree depth heuristic**

In the structure-mapper agent prompt, add after the "annotated directory tree" instruction:
```
Limit directory tree to 3 levels deep, ~30 entries max. For monorepos, show package/workspace boundaries and note the count of sub-packages.
```

Also update the Edge Cases table:
- Change "Limit directory tree to 2 levels deep" (line 411) → "3 levels deep, ~30 entries max"
- Change "Limit to 2 levels of nesting" for monorepos (line 408) → "3 levels"
- On the same monorepo row (line 408), replace `(package.json, Cargo.toml, go.mod, etc.)` with `(package.json, Cargo.toml, go.mod, and other manifest files for the detected language)`

Update the embedded command file section with the same changes.

- [ ] **Step 4: Fix #16 — Add adaptive import detection for incremental updates**

In the "Diff-Aware Scoping Logic" section (after the `git diff -G` regex around line 400), add:
```
For languages not covered by the regex heuristic above, detect changed imports by checking if the diff for each modified file includes changes to lines matching the language-appropriate import patterns identified during the initial mapping. If the language is unknown, treat any change in the first 50 lines of a source file as a potential import change.
```

- [ ] **Step 5: Fix #18 — Add review-config scope note**

Near line 480 (in "Updated `/adversarial-review` Integration"), add:
```
Note: `/map` intentionally analyzes the full project regardless of `.claude/review-config.md` ignore patterns. The review-config is consumed only by `/adversarial-review`.
```

Also add this note to the embedded command file's error handling section.

- [ ] **Step 6: Update File Locations table**

Change the File Locations table (around line 488) to reflect plugin-based paths:
- `/map` command: `commands/map.md` (in the claude-code-commands plugin repo)
- `/adversarial-review` command: `commands/adversarial-review.md` (in the claude-code-commands plugin repo)

- [ ] **Step 7: Commit Groups 2–4 fixes for SPEC-map.md**

```bash
git add docs/specs/SPEC-map.md
git commit -m "spec(map): apply cross-file alignment, structural, and minor fixes"
```

---

## Chunk 3: Update SPEC-adversarial-review.md

### Task 5: Apply Group 1 fixes to SPEC-adversarial-review.md (adaptive discovery)

**Files:**
- Modify: `docs/specs/SPEC-adversarial-review.md`

Reference: Design spec "The Hybrid Adaptive Pattern" → "Applied Areas" section. Use the SAME example sets applied to SPEC-map.md in Task 3.

- [ ] **Step 1: Update Phase 1 — project instruction files (line 56)**

Replace:
```
Project instruction files: `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `COPILOT.md`, or similar
```
With:
```
Project instruction files: `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `COPILOT.md`, `CURSOR.md`, `AIDER.md`, `.cursorrules`, `.windsurfrules`, `.clinerules`, `.github/copilot-instructions.md`, or any root-level markdown or dotfile that appears to contain AI assistant instructions
```

- [ ] **Step 2: Update Phase 1 — specs and plans (line 57)**

Keep as-is (already generic: "glob for `*.md` in root").

- [ ] **Step 3: Update Phase 1 — package manifests (line 58)**

Replace:
```
Package manifests: `package.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`, `Gemfile`, `pom.xml`, or equivalent
```
With:
```
Package manifests (examples, not exhaustive): `package.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`, `mix.exs`, `build.sbt`. Also check the project root for any other build, package, or dependency files specific to the detected language.
```

- [ ] **Step 4: Update Phase 1 — project config (line 59)**

Replace:
```
Project config: `tsconfig.json`, `vitest.config.ts`, `jest.config.*`, `webpack.config.*`, `.eslintrc.*`, or equivalent
```
With:
```
Project config (examples, not exhaustive): `.eslintrc.*` (JS), `rustfmt.toml` (Rust), `.rubocop.yml` (Ruby), `mypy.ini` (Python), `.golangci.yml` (Go), `.formatter.exs` (Elixir). Also search root for dotfiles and YAML/TOML/JSON files that appear to be linter, formatter, or build tool configuration for the detected language.
```

- [ ] **Step 5: Update Phase 1 — hook scripts and CI (line 60)**

Replace:
```
Hook scripts and CI config: glob `scripts/**/*`, `.github/**/*`, or similar automation
```
With:
```
Hook scripts and CI config: glob `scripts/**/*`, `.github/workflows/**/*`, `.gitlab-ci.yml`, `.circleci/**/*`, `Jenkinsfile*`, `.buildkite/**/*`, `.travis.yml`, `azure-pipelines.yml`, `bitbucket-pipelines.yml`. Check for any other CI/automation configuration present.
```

- [ ] **Step 6: Update Phase 1 — source and tests (line 61)**

Replace:
```
Source and tests: glob `src/**/*` and `tests/**/*` (or `test/**/*`, `spec/**/*`, `__tests__/**/*`)
```
With:
```
Source and tests: Run `ls` on the project root and check the manifest to discover actual source and test directories. Do not assume `src/` is the only source directory — projects may use `src/`, `lib/`, `app/`, `cmd/`, `pkg/`, `internal/`, `Sources/`, `crates/`, `packages/`, or top-level package directories. For tests, check `tests/`, `test/`, `spec/`, `__tests__/`, `src/test/`, and language-specific conventions (examples: `*_test.go`, `test_*.py`, `*_spec.rb`, `t/*.t`). For any other language, check for its test naming conventions. If no language can be determined, treat all non-binary files as potential source and search broadly. Note the uncertainty in the output.
```

- [ ] **Step 7: Add NEW infrastructure/build discovery bullet (fix #11)**

After the source/tests bullet, add a new bullet:
```
Infrastructure and build: `Dockerfile*`, `docker-compose*`, `.env*`, `Makefile`
```

- [ ] **Step 8: Search for and replace ALL remaining catch-alls**

Search SPEC-adversarial-review.md for every instance of `etc.`, `or similar`, `or equivalent` **in discovery lists and agent prompts**. Replace each with the appropriate adaptive instruction. Instances in output template text (e.g., summary bullet templates, example output formatting) are acceptable to keep as user-facing prose. The PM agent prompt (around line 207) contains `(CLAUDE.md, AGENTS.md, etc.)` — replace with the full AI assistant file list. Verify zero discovery/prompt catch-alls remain.

- [ ] **Step 9: Commit Group 1 fixes for SPEC-adversarial-review.md**

```bash
git add docs/specs/SPEC-adversarial-review.md
git commit -m "spec(adversarial-review): apply hybrid adaptive pattern to all discovery lists"
```

---

### Task 6: Apply Groups 2–4 fixes to SPEC-adversarial-review.md

**Files:**
- Modify: `docs/specs/SPEC-adversarial-review.md`

- [ ] **Step 1: Fix #2 — Update "Impact Map" references to "Change Impact Map"**

Search the file for "Impact Map" and replace with "Change Impact Map" to match SPEC-map.md's heading. This appears around line 19, line 48, and in the integration table.

- [ ] **Step 2: Fix #9 — Add subagent_type to Phase 2 agents**

At line 86, change:
```
Launch **three reviewer agents in parallel** using separate Agent tool calls in a single response.
```
To:
```
Launch **three reviewer agents in parallel** using separate Agent tool calls (subagent_type: "general-purpose") in a single response.
```

- [ ] **Step 3: Fix #7 — Add --diff fallback warning**

At line 67 (Phase 1 --diff behavior), after the `HEAD~1` fallback, add:
```
Print a warning to the user: "No valid `/map` baseline found. Diffing against HEAD~1 only. Consider running `/map` first for a comprehensive baseline."
```

At line 306 (Diff-Aware Review section), add the same warning text after the `HEAD~1` fallback description.

- [ ] **Step 4: Fix #10 — Add error handling section**

Add a new section after the Edge Cases table (around line 341):

```markdown
## Error Handling

| Condition | Behavior |
|-----------|----------|
| Not a git repo | Print warning: "Not a git repository. Running full review (--diff ignored if present)." Proceed without diff capability. |
| Git unavailable | Print warning: "Git not found. Running full review (--diff ignored if present)." Proceed without diff capability. |
| Shallow clone | Print warning: "Shallow clone detected. Diff history may be limited." Proceed with available history. |
| `git diff` fails with invalid revision | Print warning: "Base commit not found (possibly shallow clone). Running full review." Ignore --diff. |
| Single reviewer agent fails | Retry once. If it fails again, proceed with findings from the other reviewers. Note which reviewer was skipped. |
```

- [ ] **Step 5: Fix #13 — Add large repo guidance**

In Phase 2 (around line 86), add after the agent launch instruction:
```
If the discovered file list exceeds 50 files, pass at most 30 file paths to each reviewer agent. Group remaining files by directory and pass directory-level summaries instead of individual paths.
```

- [ ] **Step 6: Fix #17 — Generalize BDD-specific bullets in coverage-reviewer**

In the coverage-reviewer agent prompt (around lines 150-155), replace these two bullets:
```
- Gherkin scenarios that violate the test framework's requirements
- Test lifecycle patterns that conflict with the BDD framework
```
With this single bullet:
```
- Check for specialized testing paradigms (BDD/Gherkin `.feature` files, property-based testing, snapshot testing, contract testing) and validate against their framework requirements. For all other tests, evaluate against the project's actual testing patterns.
```

Additionally, apply these two polyglot generalizations (beyond finding #17, but required for project-agnostic coverage review):

Replace:
```
- Exported functions/classes missing test behaviors (will break 100% coverage)
```
With:
```
- Exported functions/classes that appear to lack test coverage (check the project's coverage config for threshold)
```

Replace:
```
- Computation logic that is specified in types but never defined in behaviors
```
With:
```
- Interfaces, abstract definitions, or contracts that declare behavior but lack implementations or tests. Skip if the language doesn't use explicit type/interface declarations.
```

- [ ] **Step 7: Update File Locations table**

Change the File Locations table (around line 375) to reflect plugin-based paths:
- `/adversarial-review` command: `commands/adversarial-review.md` (in the claude-code-commands plugin repo)

- [ ] **Step 8: Commit Groups 2–4 fixes for SPEC-adversarial-review.md**

```bash
git add docs/specs/SPEC-adversarial-review.md
git commit -m "spec(adversarial-review): apply cross-file alignment, structural, and minor fixes"
```

---

## Chunk 4: Generate Command Files

### Task 7: Generate commands/map.md from spec

**Files:**
- Create: `commands/map.md`

- [ ] **Step 1: Extract the "Command File" section from SPEC-map.md**

Read the updated SPEC-map.md. Find the section starting with `## Command File: map.md` (around line 495). The content between the outer ```` ````markdown ```` fence is the full command file. Extract it and write to `commands/map.md`.

- [ ] **Step 2: Verify the command file includes all fixes**

Check that `commands/map.md` contains:
- Adaptive manifest discovery (not ending in `etc.`)
- Adaptive import patterns with 6+ language examples
- Adaptive test globs (no `*.*` patterns)
- Adaptive linter/config discovery
- Adaptive CI config list (comprehensive, `**/*` not `**/*.*`)
- AI assistant file comprehensive list
- Tree depth heuristic (3 levels, ~30 entries)
- Shallow clone error handling
- review-config scope note

- [ ] **Step 3: Commit**

```bash
git add commands/map.md
git commit -m "feat: generate commands/map.md from updated spec"
```

---

### Task 8: Generate commands/adversarial-review.md from spec

**Files:**
- Create: `commands/adversarial-review.md`

Note: SPEC-adversarial-review.md does NOT have an embedded "Command File" section like SPEC-map.md does. The command file must be synthesized from the spec's behavioral descriptions. Read the current command file at `~/.claude/commands/adversarial-review.md` as a starting point, then apply all the spec updates to produce the new version.

- [ ] **Step 1: Read the current command file and the updated spec**

Read both:
- `~/.claude/commands/adversarial-review.md` (current working version with partial fixes)
- `docs/specs/SPEC-adversarial-review.md` (updated source of truth)

- [ ] **Step 2: Generate the new command file**

Write `commands/adversarial-review.md` incorporating all spec updates:
- Phase 1 discovery with all adaptive patterns
- Phase 2 with `subagent_type: "general-purpose"` on reviewer agents
- Phase 2 with large repo file limit guidance
- Updated coverage-reviewer prompt (generalized testing paradigms)
- `--diff` fallback warning
- Error handling section
- All catch-alls replaced with adaptive language

- [ ] **Step 3: Verify the command file includes all fixes**

Check that `commands/adversarial-review.md` contains:
- Adaptive discovery for all 7 areas (manifests, config, tests, imports, source dirs, CI, AI assistant files)
- Infrastructure/build discovery bullet
- `subagent_type: "general-purpose"` on Phase 2 agents
- "Change Impact Map" (not "Impact Map")
- `--diff` fallback warning text
- Error handling section (git unavailable, shallow clone, non-git)
- Large repo guidance (50 file limit, 30 paths per agent)
- Generalized testing paradigm check (not BDD-specific)
- No `etc.`, `or similar`, or `or equivalent` as sole catch-alls

- [ ] **Step 4: Commit**

```bash
git add commands/adversarial-review.md
git commit -m "feat: generate commands/adversarial-review.md from updated spec"
```

---

## Chunk 5: Install and Verify

### Task 9: Install plugin and verify

**Files:**
- Delete: `~/.claude/commands/map.md`
- Delete: `~/.claude/commands/adversarial-review.md`

- [ ] **Step 1: Install the plugin**

```bash
git clone https://github.com/belchman/claude-code-commands
claude plugins install ./claude-code-commands
```

Expected: "Successfully installed plugin: claude-code-commands"

- [ ] **Step 2: Verify plugin is in settings**

```bash
cat ~/.claude/settings.json
```

Expected: `"claude-code-commands@..."` appears in `enabledPlugins`.

- [ ] **Step 3: Verify commands are available**

Start a new Claude Code session and check that `/map` and `/adversarial-review` are listed as available commands.

- [ ] **Step 4: Delete old command files**

Only after steps 1-3 succeed:

```bash
rm ~/.claude/commands/map.md
rm ~/.claude/commands/adversarial-review.md
```

- [ ] **Step 5: Final verification**

Start a fresh Claude Code session in a test project. Run:
- `/map` — verify it generates ARCHITECTURE.md
- `/adversarial-review` — verify it discovers files and launches reviewers

- [ ] **Step 6: Commit any final adjustments**

If any issues were found during verification, fix them and commit.
