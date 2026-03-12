# Spec: `/adversarial-review` — Adversarial Review Skill

> **Purpose**: A user-level Claude Code slash command that orchestrates parallel adversarial reviewers to find gaps, contradictions, and missing pieces in a project's documentation, configuration, and code — then fixes what matters.
>
> **Location**: `~/.claude/commands/adversarial-review.md`
> **Consumes**: `ARCHITECTURE.md` (from `/map`), project-level overrides from `.claude/review-config.md`

---

## Overview

`/adversarial-review` discovers a project's documentation, configuration, and code, then launches three domain-specific reviewer agents in parallel. Findings are merged, presented to the user, and optionally fixed by an adversarial PM agent.

The skill is **project-agnostic** — it auto-discovers what exists and adapts its review scope accordingly. When `ARCHITECTURE.md` exists (from `/map`), it uses the Change Impact Map and dependency graph to prioritize high-risk findings.

---

## Invocation

```
/adversarial-review                    # Review all discovered files
/adversarial-review SPEC.md            # Review specific file(s) + context
/adversarial-review --diff             # Review only changes since last commit
/adversarial-review --diff SPEC.md     # Review changes in specific file(s) since last commit
```

### Arguments: `$ARGUMENTS`

| Argument | Default | Description |
|----------|---------|-------------|
| (none) | — | Auto-discover and review all project documentation and configuration |
| `<file path or glob>` | — | Review specific files as primary targets, still discover surrounding context |
| `--diff` | — | Scope review to changes since last commit (or since `ARCHITECTURE.md`'s `last-mapped` SHA if available) |

### Parsing

`$ARGUMENTS` is split on whitespace. `--diff` can appear anywhere in the arguments. Remaining non-flag arguments are treated as file paths or globs. If `--diff` is used but the repo has no commits, fall back to full review and warn the user.

---

## Behavior

### Phase 1: Discovery

Before launching any agents, the orchestrator must understand the project. Steps:

1. **Check for `ARCHITECTURE.md`** at the repo root.
   - If it exists: read it and extract the Change Impact Map, Dependencies (Mermaid graph), and Conventions sections. These provide review context.
   - If it does not exist: print a suggestion — "No ARCHITECTURE.md found. Consider running `/map` first for richer context." Then proceed with manual discovery.

2. **Check for project-level overrides** at `.claude/review-config.md`.
   - If it exists: read it and apply any custom reviewer focus areas, files to ignore, or severity overrides. See [Review Config](#review-config) section.
   - If it does not exist: proceed with defaults.

3. **Auto-discover project files** (regardless of whether ARCHITECTURE.md exists):
   - Project instruction files: `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `COPILOT.md`, `CURSOR.md`, `AIDER.md`, `.cursorrules`, `.windsurfrules`, `.clinerules`, `.github/copilot-instructions.md`, or any root-level markdown or dotfile that appears to contain AI assistant instructions
   - Specs and plans: glob for `*.md` in root — any file that looks like a specification, PRD, plan, or architecture doc
   - Package manifests (examples, not exhaustive): `package.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`, `mix.exs`, `build.sbt`. Also check the project root for any other build, package, or dependency files specific to the detected language.
   - Project config (examples, not exhaustive): `.eslintrc.*` (JS), `rustfmt.toml` (Rust), `.rubocop.yml` (Ruby), `mypy.ini` (Python), `.golangci.yml` (Go), `.formatter.exs` (Elixir). Also search root for dotfiles and YAML/TOML/JSON files that appear to be linter, formatter, or build tool configuration for the detected language.
   - Hook scripts and CI config: glob `scripts/**/*`, `.github/workflows/**/*`, `.gitlab-ci.yml`, `.circleci/**/*`, `Jenkinsfile*`, `.buildkite/**/*`, `.travis.yml`, `azure-pipelines.yml`, `bitbucket-pipelines.yml`. Check for any other CI/automation configuration present.
   - Source and tests: Run `ls` on the project root and check the manifest to discover actual source and test directories. Do not assume `src/` is the only source directory — projects may use `src/`, `lib/`, `app/`, `cmd/`, `pkg/`, `internal/`, `Sources/`, `crates/`, `packages/`, or top-level package directories. For tests, check `tests/`, `test/`, `spec/`, `__tests__/`, `src/test/`, and language-specific conventions (examples: `*_test.go`, `test_*.py`, `*_spec.rb`, `t/*.t`). For any other language, check for its test naming conventions. If no language can be determined, treat all non-binary files as potential source and search broadly. Note the uncertainty in the output.
   - Infrastructure and build: `Dockerfile*`, `docker-compose*`, `.env*`, `Makefile`

4. **If user passed specific files**: those become the primary review targets. Still discover surrounding context so reviewers understand the full picture.

5. **If `--diff` was passed**: scope the review.
   - If `ARCHITECTURE.md` exists with a `last-mapped` SHA: use `git diff <last-mapped-sha>..HEAD --name-status` to find changed files since last map.
   - Otherwise: use `git diff HEAD~1..HEAD --name-status` for committed changes + `git diff HEAD --name-status` for uncommitted changes. Print a warning to the user: "No valid `/map` baseline found. Diffing against HEAD~1 only. Consider running `/map` first for a comprehensive baseline."
   - Filter the discovered files to only include changed files and their direct dependents (files that import them, identified from ARCHITECTURE.md's dependency graph if available).
   - Always include `CLAUDE.md` and any project instruction files in the review context (even if unchanged) since they're needed for cross-referencing.

6. **Summarize what was found** in 3-5 bullets for the user before proceeding:
   - What kind of project this is (language, framework, purpose)
   - Which files will be the primary review targets
   - What supporting context was found (ARCHITECTURE.md, review-config, etc.)
   - If `--diff`: which files changed and how many dependents were included

7. **Classify discovered files into three domains** for the parallel review:
   - **Spec domain**: specs, plans, PRDs, architecture docs, type definitions
   - **Config domain**: settings files, hook scripts, CI config, permissions, env vars
   - **Coverage domain**: test behaviors/requirements, coverage thresholds, test framework config, feature files

---

### Phase 2: Parallel Adversarial Review

Launch **three reviewer agents in parallel** using separate Agent tool calls (subagent_type: "general-purpose") in a single response. Each agent focuses on one domain but reads ALL files for cross-reference context.

If the discovered file list exceeds 50 files, pass at most 30 file paths to each reviewer agent. Group remaining files by directory and pass directory-level summaries instead of individual paths.

**IMPORTANT: Launch all three agents simultaneously in a single response.**

If `ARCHITECTURE.md` exists, include this context preamble in every agent prompt:

```
## Project Context (from ARCHITECTURE.md)

### High-Coupling Zones (changes here have wide blast radius):
[paste the High-Coupling Zones table]

### Key Interfaces:
[paste the Key Interfaces list]

### Conventions:
[paste the Conventions section]

Use this context to:
- Flag findings as higher severity when they affect high-coupling zones
- Cross-reference dependency relationships when checking for contradictions
- Evaluate code/config against established conventions
```

#### Agent 1: `spec-reviewer` — Spec Consistency

```
You are an adversarial SPEC CONSISTENCY reviewer. Focus on:
- Internal contradictions within spec/plan documents
- Cross-references between docs that don't match (phases, numbering, naming)
- Missing definitions referenced elsewhere
- Ambiguities where an AI agent could go either way
- Prose behaviors that contradict code examples
- Type definitions that don't match what tests reference

[PROJECT CONTEXT from ARCHITECTURE.md if available]

Read ALL files: [LIST ALL DISCOVERED FILES WITH ABSOLUTE PATHS]

Return ONLY findings as numbered markdown. Each finding must have: number, file+line reference, what's wrong, evidence (quoted text), suggested fix. Categorize as CRITICAL / IMPORTANT / MINOR.
```

#### Agent 2: `config-reviewer` — Config & Hooks

```
You are an adversarial CONFIG & HOOKS reviewer. Focus on:
- Hook scripts that won't work for the file paths described in specs
- Permission gaps (commands the spec requires but permissions don't allow)
- Settings that contradict documented behavior
- Hook logic errors (wrong grep patterns, path resolution issues, race conditions)
- Environment variables that are missing or wrong
- Slash commands that reference paths or behaviors incorrectly

[PROJECT CONTEXT from ARCHITECTURE.md if available]

Read ALL files: [LIST ALL DISCOVERED FILES WITH ABSOLUTE PATHS]

Return ONLY findings as numbered markdown. Each finding must have: number, file+line reference, what's wrong, evidence (quoted text), suggested fix. Categorize as CRITICAL / IMPORTANT / MINOR.
```

#### Agent 3: `coverage-reviewer` — Test Coverage & Completeness

```
You are an adversarial TEST COVERAGE reviewer. Focus on:
- Exported functions/classes that appear to lack test coverage (check the project's coverage config for threshold)
- Test behaviors that can't be implemented as described
- Check for specialized testing paradigms (BDD/Gherkin `.feature` files, property-based testing, snapshot testing, contract testing) and validate against their framework requirements. For all other tests, evaluate against the project's actual testing patterns.
- Coverage config that excludes too much or too little
- Interfaces, abstract definitions, or contracts that declare behavior but lack implementations or tests. Skip if the language doesn't use explicit type/interface declarations.

[PROJECT CONTEXT from ARCHITECTURE.md if available]

Read ALL files: [LIST ALL DISCOVERED FILES WITH ABSOLUTE PATHS]

Return ONLY findings as numbered markdown. Each finding must have: number, file+line reference, what's wrong, evidence (quoted text), suggested fix. Categorize as CRITICAL / IMPORTANT / MINOR.
```

**After all three agents finish:**
1. Merge all findings into a single report
2. De-duplicate (if two reviewers found the same issue, keep the more detailed one)
3. Re-number findings sequentially
4. If ARCHITECTURE.md context was available, annotate findings that affect high-coupling zones with a `[HIGH RISK]` tag
5. Print the merged report for the user

---

### Phase 3: User Checkpoint

After showing the merged findings, ask the user:

> Here are the findings from 3 parallel reviewers. Which should I fix?
> - **A)** All critical + important findings
> - **B)** Only critical findings
> - **C)** Let me pick specific ones (list the finding numbers)
> - **D)** Skip fixes — I just wanted the review

**Wait for the user to respond before proceeding.** If they choose D, stop here.

---

### Phase 4: Adversarial Product Manager Agent

Launch an agent with `subagent_type: "general-purpose"` named `adversarial-pm`.

Build the prompt dynamically. Include the full text of every selected finding. The template:

```
You are an **adversarial product manager**. You have a list of findings from a technical review. Your job is to make the **minimum** edits to fix each finding. Nothing more.

**Read the current state of every file you're about to edit BEFORE making changes.**

**Findings to fix:**
[PASTE THE FULL TEXT OF EVERY SELECTED FINDING HERE]

**General rules:**
- Fix exactly what the finding describes. No scope creep.
- If a section contradicts another, fix the one that's wrong (not both).
- If something is missing, add it in the most logical location near related content.
- Match the existing style, formatting, and voice of each file.

**Rules for project instruction files (`CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `COPILOT.md`, `CURSOR.md`, `AIDER.md`, `.cursorrules`, `.windsurfrules`, `.clinerules`, `.github/copilot-instructions.md`, or any root-level markdown or dotfile containing AI assistant instructions):**
- **ONLY ADD content, never remove** unless something is factually wrong or directly contradicts the spec/plan.
- If a section needs updating, append or clarify — don't rewrite.
- If nothing needs changing, don't touch it.

**Rules for specs, plans, and documentation:**
- Fix exactly what the finding identifies.
- Preserve the document's structure and organization.
- If fixing a contradiction, keep the version that's consistent with the rest of the document.

**Rules for config files:**
- Only change if a finding specifically identifies a config issue.
- Never change permissions, hooks, or security settings unless explicitly flagged as a finding.

**After each file edit, state:**
1. What finding this addresses (by number)
2. What you changed (1 sentence)
3. Why this is the minimum fix

**Do NOT:**
- Add features or new sections not identified in findings
- Refactor or reorganize existing content
- Add comments explaining your changes inside the files
- Touch files that aren't mentioned in the findings
```

**Wait for the PM agent to finish.** Then summarize what was changed for the user.

---

### Phase 5: Summary

After the PM agent finishes, present a summary:

```
## Adversarial Review Complete

### Reviewers: 3 parallel agents (spec, config, coverage)
### Findings: X critical, Y important, Z minor
### Fixed: N findings across M files

### Changes made:
- [file]: [1-line description of change] (finding #N)
- ...

### Not fixed (if any):
- Finding #N: [reason — user chose to skip, out of scope, etc.]
```

If any project instruction file (`CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `COPILOT.md`, `CURSOR.md`, `AIDER.md`, `.cursorrules`, `.windsurfrules`, `.clinerules`, `.github/copilot-instructions.md`, or any other AI assistant instruction file) was modified, call it out explicitly so the user can review those changes.

Suggest running `/adversarial-review` again if critical fixes were made, to verify no regressions.

---

## Review Config

Projects can customize review behavior via `.claude/review-config.md`. This file is optional and uses a simple markdown format:

```markdown
# Review Config

## Focus Areas
- Pay special attention to API contract consistency
- Check that all error codes are documented

## Ignore
- docs/archive/**
- generated/**
- *.min.js

## Severity Overrides
- Missing JSDoc on exported functions: MINOR (not IMPORTANT)
- Inconsistent naming in test files: MINOR
```

### Sections

| Section | Purpose |
|---------|---------|
| `## Focus Areas` | Additional review priorities added to all agent prompts |
| `## Ignore` | File globs to exclude from review (one per line) |
| `## Severity Overrides` | Rules to reclassify finding severity (format: `description: SEVERITY`) |

If the file exists, the orchestrator:
1. Reads it during Phase 1
2. Appends Focus Areas to each reviewer agent's prompt
3. Filters discovered files against Ignore patterns before passing to agents
4. Passes Severity Overrides to the merge step in Phase 2 for re-classification

---

## Diff-Aware Review (`--diff`)

When `--diff` is passed, the review is scoped to recent changes:

1. **Determine the base reference**:
   - If `ARCHITECTURE.md` exists with `last-mapped` SHA: use that SHA as the base
   - If `ARCHITECTURE.md` exists but `last-mapped` is `0000000`: fall back to full review
   - Otherwise: use `HEAD~1` as the base (last commit). Print a warning to the user: "No valid `/map` baseline found. Diffing against HEAD~1 only. Consider running `/map` first for a comprehensive baseline."

2. **Gather changed files**:
   - `git diff <base>..HEAD --name-status` for committed changes
   - `git diff HEAD --name-status` for uncommitted changes
   - `git ls-files --others --exclude-standard` for untracked files (status `A`)
   - Merge results (uncommitted status wins on conflicts)

3. **Expand to dependents** (if ARCHITECTURE.md's dependency graph is available):
   - For each changed file, find files that import it (from the Mermaid graph or JSON data)
   - Include those dependent files in the review scope
   - This catches "blast radius" issues where a change in one file breaks consumers

4. **Always include in review context** (even if unchanged):
   - `CLAUDE.md`, `AGENTS.md`, and other project instruction files
   - `ARCHITECTURE.md` itself
   - `.claude/review-config.md` if it exists

5. **Pass scoping context to reviewers**:
   - Each reviewer agent gets a preamble: "This is a diff-scoped review. Focus on changes in the following files: [list]. Also review these context files for cross-reference: [list]. Flag issues in changed files at normal severity. Flag issues in context files only if they directly contradict the changed files."

---

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| No documentation files found | Print warning: "No specs, plans, or documentation found. Review will focus on config and code structure only." Skip the spec-reviewer agent. |
| No config files found | Skip the config-reviewer agent. |
| No test files found | The coverage-reviewer still runs but focuses on whether tests should exist (missing coverage). |
| No source files at all | Print warning: "No source files found. This appears to be a documentation-only project." Run only the spec-reviewer. |
| `--diff` with no changes | Print "No changes detected since last review baseline." and stop. |
| `--diff` with no git repo | Print warning: "Not a git repository. Running full review." Ignore `--diff`. |
| All three reviewers return zero findings | Print "No findings — the project looks consistent." and stop. |
| Single reviewer fails | Retry once. If it fails again, proceed with findings from the other two reviewers and note which reviewer was skipped. |

---

## Error Handling

| Condition | Behavior |
|-----------|----------|
| Not a git repo | Print warning: "Not a git repository. Running full review (--diff ignored if present)." Proceed without diff capability. |
| Git unavailable | Print warning: "Git not found. Running full review (--diff ignored if present)." Proceed without diff capability. |
| Shallow clone | Print warning: "Shallow clone detected. Diff history may be limited." Proceed with available history. |
| `git diff` fails with invalid revision | Print warning: "Base commit not found (possibly shallow clone). Running full review." Ignore --diff. |
| Single reviewer agent fails | Retry once. If it fails again, proceed with findings from the other reviewers. Note which reviewer was skipped. |

---

## Integration with `/map`

When `ARCHITECTURE.md` exists, the review is enhanced:

| ARCHITECTURE.md Section | How It's Used |
|------------------------|---------------|
| **Change Impact Map → High-Coupling Zones** | Findings affecting these files get `[HIGH RISK]` annotation. Reviewers are told to scrutinize changes to these files more carefully. |
| **Dependencies → Mermaid Graph** | Used in `--diff` mode to expand review scope to dependent files. Also used by spec-reviewer to verify that documented interfaces match implementations. |
| **Conventions** | Included in reviewer context so they can flag code that violates established patterns. |
| **Structure** | Helps reviewers understand the project layout without re-discovering it. |

---

## Success Criteria

1. Running `/adversarial-review` on a fresh project auto-discovers files and produces findings
2. Running `/adversarial-review SPEC.md` scopes the primary review to that file while including context
3. Running `/adversarial-review --diff` reviews only recent changes and their dependents
4. Three reviewer agents always launch in parallel (single response with 3 Agent calls)
5. Findings are merged, de-duplicated, and presented with severity categories
6. The user checkpoint always happens before any fixes are made
7. The PM agent makes minimum edits that address findings precisely
8. `ARCHITECTURE.md` context enriches findings when available but is not required
9. `.claude/review-config.md` overrides are respected when present

---

## File Locations

| File | Location | Committed to git? |
|------|----------|-------------------|
| `/adversarial-review` command | `commands/adversarial-review.md` (in the project-tools plugin repo) | No (user-level) |
| Review config overrides | `<repo-root>/.claude/review-config.md` | Yes (optional) |
| `ARCHITECTURE.md` (consumed) | `<repo-root>/ARCHITECTURE.md` | Yes (from `/map`) |
