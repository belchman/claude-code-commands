# Design: `project-tools` Plugin — Polyglot-Adaptive Claude Code Commands

> **Purpose**: Convert `/map` and `/adversarial-review` from loose user-level commands into an installable Claude Code plugin, applying a hybrid adaptive pattern to make them fully project-agnostic.
>
> **Repo**: `~/projects/claude-skills/`
> **Plugin name**: `project-tools`
> **Approach**: Hybrid — short example lists + mandatory adaptive fallback (Approach 3)

---

## Context

Two slash commands (`/map`, `/adversarial-review`) exist as user-level files in `~/.claude/commands/`. They work but have 18 known issues from adversarial review, primarily:

- Static file/pattern lists biased toward JS/TS/Python/Rust/Go ecosystem
- Cross-file misalignment (different CI globs, different example sets, naming inconsistencies)
- Structural gaps (missing error handling, no section heading mapping, vague limits)

These commands should work for **any language**, including uncommon ones (Haskell, Scala, Zig, Nim, OCaml, COBOL, etc.). The fix strategy is a hybrid adaptive pattern where explicit examples illustrate but don't limit, and an adaptive fallback is the primary mechanism.

---

## Repo Structure

```
~/projects/claude-skills/
├── .claude-plugin/
│   └── plugin.json              # Plugin metadata (name: "project-tools")
├── commands/
│   ├── map.md                   # /map command (executable)
│   └── adversarial-review.md    # /adversarial-review command (executable)
├── docs/
│   └── specs/
│       ├── SPEC-map.md                              # Detailed spec (source of truth)
│       ├── SPEC-adversarial-review.md               # Detailed spec (source of truth)
│       └── 2026-03-12-project-tools-plugin-design.md # This document
├── .gitignore
├── LICENSE
└── README.md
```

### Spec-to-Command Relationship

Specs in `docs/specs/` are the **source of truth**. Command files in `commands/` are the **executable output**. Each spec contains a "Command File" section with the full markdown that goes into the corresponding command file.

Future changes follow: **edit spec → regenerate command file to match**.

---

## The Hybrid Adaptive Pattern

### Structure

Every static file/pattern list in both command files gets restructured to a three-part format:

```
1. DETECT: "First identify the project's language(s) from manifests and file extensions."
2. EXAMPLES: Short diverse list (~5-6 languages) explicitly labeled as "examples, not exhaustive"
3. ADAPT: "For any language not listed, discover its conventions by [specific technique]."
```

This replaces all `etc.`, `or similar`, `or equivalent` catch-alls with actionable adaptive instructions. The examples anchor common cases but the adapt step is the primary mechanism.

### Undetectable Language Fallback

If no language can be determined (no manifest, no recognizable extensions — e.g., custom DSLs, polyglot scripts, generated code), the agent must not silently skip discovery. Explicit fallback:

> "If no language can be determined from manifests or file extensions, treat all non-binary files as potential source code. Search broadly for common patterns: look for files with import-like statements, test-like naming conventions, and config-like dotfiles. Note the uncertainty in the output."

### Applied Areas

Both command files use the **same** example sets (eliminating cross-file alignment issues):

**Manifests** (detect → examples → adapt):
- Examples: `package.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`, `mix.exs`, `build.sbt`
- Adapt: "Check the project root for any other build, package, or dependency files"

**Config/linters** (detect → examples → adapt):
- Examples: `.eslintrc.*` (JS), `rustfmt.toml` (Rust), `.rubocop.yml` (Ruby), `mypy.ini` (Python), `.golangci.yml` (Go), `.formatter.exs` (Elixir)
- Adapt: "Search root for dotfiles and YAML/TOML/JSON files that appear to be linter, formatter, or build tool configuration for the detected language"

**Test patterns** (detect → examples → adapt):
- Examples: `tests/`, `test/`, `spec/`, `__tests__/`, `*_test.go`, `test_*.py`, `*_spec.rb`, `t/*.t`
- Adapt: "Check for test file naming conventions specific to the detected language. When in doubt, search for files importing test framework modules"

**Import patterns** (detect → examples → adapt):
- Examples: `import`/`from` (Python/JS/TS/Java/Go/Swift/Dart), `require`/`require_relative` (Ruby/Node), `use`/`mod` (Rust), `using` (C#), `#include` (C/C++), `open` (OCaml)
- Adapt: "For any language not listed, research its module/import system and grep for the appropriate keywords. Only search source files, not comments or documentation"

**Source directories** (detect → examples → adapt):
- Examples: `src/`, `lib/`, `app/`, `pkg/`, `cmd/`, `internal/`, `Sources/`, `crates/`, `packages/`
- Adapt: "Do NOT limit yourself to these examples. Run `ls` on the root first, then explore ALL directories that appear to contain source code. Do not assume `src/` is the only source directory"

**CI config** (detect → examples → adapt):
- Examples: `.github/workflows/**/*`, `.gitlab-ci.yml`, `.circleci/**/*`, `Jenkinsfile*`, `.buildkite/**/*`, `.travis.yml`, `azure-pipelines.yml`, `bitbucket-pipelines.yml`
- Adapt: "Check for any other CI/automation configuration present in the project"

**AI assistant instructions** (explicit list + catch-all):
- List: `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `COPILOT.md`, `CURSOR.md`, `AIDER.md`, `.cursorrules`, `.windsurfrules`, `.clinerules`, `.github/copilot-instructions.md`
- Catch-all: "or any root-level markdown or dotfile that appears to contain AI assistant instructions"

---

## Fix Groups

### Group 1: Adaptive Discovery Pattern (findings #3, #4, #5, #6)

Apply the hybrid adaptive pattern (above) to all file/pattern lists in both command files and both specs. This is the core change.

### Group 2: Cross-File Alignment (findings #1, #2, #8, #9, #14)

| Finding | Fix |
|---------|-----|
| #1 CI globs | Both files adopt `**/*` (no extension requirement). Remove `*.*` patterns. |
| #2 Section naming | `map.md` spec gets explicit heading table: `deps` → `## Dependencies`, `impact` → `## Change Impact Map`. `adversarial-review.md` references "Change Impact Map" (matching map.md spec's existing heading). |
| #8 src/ missing | Add `src/` to adversarial-review.md source directory list. Rephrase: "Do not assume `src/` is the only source directory." |
| #9 subagent_type | Add `subagent_type: "general-purpose"` to Phase 2 agent launch instructions in adversarial-review.md. |
| #14 AI assistant files | Both files adopt the same comprehensive list (see AI assistant instructions in the adaptive pattern above). |

### Group 3: Structural Gaps (findings #7, #10)

| Finding | Fix |
|---------|-----|
| #7 --diff fallback | Add explicit warning when falling back to `HEAD~1`: "No valid `/map` baseline found. Diffing against HEAD~1 only. Consider running `/map` first for a comprehensive baseline." |
| #10 Error handling | Add error handling section to adversarial-review.md covering: git unavailable (skip `--diff` with warning), shallow clones (warn about limited diff history), non-git repos (proceed without diff capability). Mirror map.md's error handling approach. |

### Group 4: Minor Fixes (findings #11–#18)

| Finding | Fix |
|---------|-----|
| #11 Docker files | Add `Dockerfile*`, `docker-compose*`, `.env*` to adversarial-review.md discovery (infrastructure/build section). |
| #12 Replace `etc.` | Replace all `etc.` in map.md agent prompts with adaptive language from the hybrid pattern. |
| #13 Large repos | Add concrete limit: "If the discovered file list exceeds 50 files, pass at most 30 file paths to each reviewer agent. Group remaining files by directory and pass directory-level summaries instead." |
| #15 Tree heuristic | Add to structure-mapper: "Limit directory tree to 3 levels deep, ~30 entries max. For monorepos, show package/workspace boundaries and note the count of sub-packages." |
| #16 Import detection | Add to incremental update section: "To detect changed imports, check if the diff for each modified file includes changes to lines matching the language-appropriate import patterns from the dependency-mapper. If the language is unknown, treat any change in the first 50 lines of a source file as a potential import change." |
| #17 BDD generalization | Replace BDD-specific callout with: "Check for specialized testing paradigms (BDD/Gherkin `.feature` files, property-based testing, snapshot testing, contract testing) and validate against their framework requirements. For all other tests, evaluate against the project's actual testing patterns." |
| #18 review-config scope | Add note to map.md: "Note: `/map` intentionally analyzes the full project regardless of `.claude/review-config.md` ignore patterns. The review-config is consumed only by `/adversarial-review`." |

---

## Mitigations for Known Failure Points

### Addressed in this plan (v1):

**F1: Undetectable languages** — Explicit fallback added to the hybrid adaptive pattern (see above). Agents must not silently skip discovery.

**F4: Stale specs vs. partially-fixed commands** — Implementation starts by reading current command files, incorporating their existing fixes into the spec updates, then regenerating commands from the updated specs. No fixes are lost.

**F5: Plugin vs. user-command precedence** — After installing the plugin, delete the old files from `~/.claude/commands/` to avoid shadowing conflicts. Verify Claude Code loads plugin commands correctly.

**F7: Large repo context overflow** — Concrete limit added: 30 file paths per reviewer agent, directory-level summaries for the rest (see finding #13 fix).

### Accepted limitations (v1):

**F2: Model knowledge ceiling** — Adaptive instructions are only as good as the model's knowledge of a given language. For truly obscure languages, coverage may be incomplete. Documented, not fixable.

**F3: No automated sync enforcement** — Both command files should use the same example sets, but there's no automated check. `/adversarial-review` can catch drift. Consider automated validation in a future version.

**F6: No automated testing** — No way to automatically verify commands work after changes. Manual testing required. Consider adding a `tests/` directory with example projects in a future version.

**F8: Import detection regex is language-specific** — The `git diff -G` regex in the spec is a heuristic. It works for common languages but may miss or false-positive for others. The spec already documents this. Accept for v1.

---

## Plugin Metadata

`.claude-plugin/plugin.json`:
```json
{
  "name": "project-tools",
  "description": "Project mapping and adversarial review commands for Claude Code — polyglot-adaptive, works on any codebase",
  "version": "1.0.0",
  "author": {
    "name": "Matthew Belchak"
  },
  "license": "MIT",
  "keywords": ["architecture", "review", "mapping", "polyglot", "project-tools"]
}
```

---

## Implementation Order

1. Set up plugin structure (`.claude-plugin/plugin.json`, directory layout)
2. Move existing specs from repo root to `docs/specs/`
3. Update SPEC-map.md with all applicable fixes (Groups 1–4) + failure mitigations
4. Update SPEC-adversarial-review.md with all applicable fixes (Groups 1–4) + failure mitigations
5. Regenerate `commands/map.md` from updated SPEC-map.md
6. Regenerate `commands/adversarial-review.md` from updated SPEC-adversarial-review.md
7. Delete old files from `~/.claude/commands/`
8. Install plugin via `claude plugins install`
9. Verify both commands work in a test project

---

## Success Criteria

1. `claude plugins install` succeeds from the repo
2. `/map` works on a JS project, a Python project, and a Go project with equivalent quality
3. `/adversarial-review` produces findings for any project type without language-specific blind spots
4. Both commands reference the same example sets and adaptive patterns
5. No `etc.`, `or similar`, or `or equivalent` remains as a sole catch-all in any agent prompt
6. Error handling covers: non-git repos, shallow clones, git unavailable, undetectable languages
7. The specs and command files are in sync
