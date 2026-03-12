# Spec: `/map` — Project Map Skill

> **Purpose**: A user-level Claude Code slash command that generates and maintains an `ARCHITECTURE.md` at each repo's root. This is the "project brain" — a persistent, living document that any skill or session can read to understand the project.
>
> **Location**: `~/.claude/commands/map.md`
> **Artifact**: `ARCHITECTURE.md` at the root of whichever repo it's run in.

---

## Overview

`/map` discovers a project's structure, dependencies, conventions, and high-coupling zones, then writes (or updates) an `ARCHITECTURE.md` file. Other skills (`/adversarial-review`, `/plan`, code reviewers) consume this artifact for context.

The skill is **project-agnostic** — it works on any language, framework, or repo structure. It discovers what exists rather than assuming a layout.

---

## Invocation

```
/map                  # Incremental update (diff-aware if ARCHITECTURE.md exists)
/map --full           # Full regeneration (preserves manual notes)
/map --section deps   # Update only the Dependencies section
```

### Arguments: `$ARGUMENTS`

| Argument | Default | Description |
|----------|---------|-------------|
| (none) | — | Incremental update if ARCHITECTURE.md exists, full generation if not |
| `--full` | — | Regenerate from scratch, preserving `<!-- manual -->` blocks |
| `--section <name>` | — | Update only one section: `structure`, `deps`, `conventions`, `impact` |

### Argument-to-Section Mapping

| Argument name | Section heading |
|---------------|-----------------|
| `structure` | `## Structure` |
| `deps` | `## Dependencies` |
| `conventions` | `## Conventions` |
| `impact` | `## Change Impact Map` |

### Parsing

`$ARGUMENTS` is split on whitespace. `--full` and `--section <name>` are mutually exclusive (if both present, `--full` takes precedence). Unknown flags are ignored with a warning. If `--section` receives an unrecognized name, print the valid options (`structure`, `deps`, `conventions`, `impact`) and exit without modifying any files.

---

## Output: `ARCHITECTURE.md` Format

The generated document has four sections plus metadata. Each section is scaled to the project's complexity — a 5-file project gets a few lines per section, a 500-file monorepo gets detailed subsections.

### Metadata Block

Hidden HTML comment at the top of the file, used for diff-aware updates:

```markdown
<!-- map-metadata
map-version: 1
last-mapped: abc1234
last-mapped-date: 2026-03-12
sections-updated: structure,deps,conventions,impact
-->
```

Fields:
- `map-version`: Format version of this metadata block (currently `1`)
- `last-mapped`: Git commit SHA at the time of last full or incremental update
- `last-mapped-date`: Human-readable date
- `sections-updated`: Which sections were touched in the last run

### Section 1: Structure

```markdown
## Structure

Brief description of the project — what it is, what it does, primary language/framework.

### Directory Layout

\`\`\`
project-root/
├── src/                    # Application source
│   ├── api/                # REST endpoints (Express routes)
│   ├── services/           # Business logic layer
│   ├── models/             # Data models and types
│   └── utils/              # Shared utilities
├── tests/                  # Test files (mirrors src/ structure)
├── scripts/                # Build and deployment scripts
├── docs/                   # Documentation
└── config/                 # Environment and app configuration
\`\`\`

### Key Entry Points

- **Application**: `src/index.ts` — Express server startup
- **CLI**: `src/cli/index.ts` — Command-line interface
- **Tests**: `npm test` runs vitest against `tests/`
```

**Discovery approach**: Glob for files, read package.json/Cargo.toml/etc., identify framework from dependencies, annotate directories by inspecting representative files.

### Section 2: Dependencies

```markdown
## Dependencies

### Module Dependency Graph

\`\`\`mermaid
graph TD
    A[src/api/routes.ts] --> B[src/services/auth.ts]
    A --> C[src/services/users.ts]
    B --> D[src/models/user.ts]
    C --> D
    C --> E[src/utils/validation.ts]
    F[src/cli/index.ts] --> C
\`\`\`

> Arrows mean "imports/depends on" — `A --> B` means A imports B. Use `graph TD` (top-down) by default.

### External Dependencies

| Package | Purpose | Used by |
|---------|---------|---------|
| express | HTTP server | src/api/* |
| prisma | Database ORM | src/models/* |
| zod | Validation | src/utils/validation.ts, src/api/* |

### Key Interfaces

> Key Interfaces describes what a module *does* (its API surface). A module may also appear in High-Coupling Zones, which describes change *risk*.

- **`UserService`** (`src/services/users.ts`) — consumed by API routes and CLI. Central interface for user operations.
- **`AuthMiddleware`** (`src/api/middleware/auth.ts`) — used by all protected routes.
```

**Discovery approach**: Grep for import/require statements, build adjacency list, identify most-imported modules (high fan-in = key interfaces). Read package.json for external deps, match to usage via grep.

### Section 3: Conventions

```markdown
## Conventions

### Patterns

- **Routing**: Each route file exports a router, mounted in `src/api/index.ts`
- **Error handling**: All services throw typed errors (`AppError`), caught by global error middleware
- **Testing**: One test file per source file, same directory structure under `tests/`
- **Naming**: camelCase for files and functions, PascalCase for classes and types

### Configuration

- **Environment**: `.env` files loaded via dotenv, validated by `src/config/index.ts`
- **TypeScript**: Strict mode, bundler resolution, ESM modules

### Established Practices

- All database queries go through Prisma models, never raw SQL
- API responses follow `{ data, error, meta }` envelope pattern
- Feature flags checked via `src/utils/flags.ts`
```

**Discovery approach**: Read 3-5 representative files from different areas, identify recurring patterns. Check linter configs, tsconfig, CI files for enforced conventions.

### Section 4: Change Impact Map

```markdown
## Change Impact Map

### High-Coupling Zones

> High-Coupling Zones describes change *risk* (blast radius based on fan-in). A module may also appear in Key Interfaces, which describes what the module *does*.

These files/modules are imported by many others. Changes here have wide blast radius:

| File | Depended on by | Risk |
|------|---------------|------|
| `src/models/user.ts` | 12 files | HIGH — any type change breaks API, services, and tests |
| `src/utils/validation.ts` | 8 files | MEDIUM — used across API and CLI layers |
| `src/config/index.ts` | 6 files | MEDIUM — env changes affect all layers |

### Critical Paths

- **Auth flow**: `routes/auth.ts` -> `services/auth.ts` -> `models/user.ts` -> database. Breaking any link breaks login.
- **API contract**: `src/types.ts` defines all API response shapes. Changes require updating tests and any consumers.

### Safe Zones

These modules are leaf nodes with no downstream dependents — safe to modify:
- `src/cli/commands/*.ts` — each command is independent
- `scripts/*` — build tooling, no runtime impact
- `docs/*` — documentation only
```

**Discovery approach**: Count import fan-in for every file. Files with fan-in >= 3 go in high-coupling. Trace call chains for critical user flows. Leaf nodes (fan-in = 0, fan-out only) are safe zones.

### Manual Notes

Users can add their own notes anywhere in the document. To protect manual additions from being overwritten during updates, wrap them in:

```markdown
<!-- manual -->
### Team-Specific Notes

- The billing module is owned by the payments team
- Never change the webhook URL format without notifying partners
<!-- /manual -->
```

The `/map` command preserves everything between `<!-- manual -->` and `<!-- /manual -->` tags during regeneration.

---

## Behavior: Execution Modes

### Mode 1: First Run (no ARCHITECTURE.md exists)

1. **Discovery** — read project files to understand language, framework, structure
2. **Parallel analysis** — launch 3 agents simultaneously using the Claude Code `Agent` tool with `subagent_type: "general-purpose"`, all three dispatched in a single response for parallelism:
   - `structure-mapper` — globs, tree, file purposes, entry points
   - `dependency-mapper` — grep imports, build dep graph, identify key interfaces
   - `convention-scanner` — read representative files, identify patterns, check configs
   If an agent fails, the orchestrator retries once. If it fails again, skip that section and warn the user.
3. **Impact analysis** — after agents complete, the orchestrator computes fan-in counts and critical paths from the dependency-mapper's structured JSON data
4. **Write** — assemble all sections into `ARCHITECTURE.md` with metadata block
5. **Summary** — print what was generated:
   ```
   Created ARCHITECTURE.md (4 sections)
     Structure: 12 directories, 3 entry points
     Dependencies: 45 modules, 8 external packages
     Conventions: 6 patterns identified
     Impact Map: 5 high-coupling zones, 3 critical paths
   ```

### Mode 2: Incremental Update (ARCHITECTURE.md exists, no `--full`)

1. **Read existing** — parse current `ARCHITECTURE.md`, extract metadata block. If `last-mapped` is `0000000`, run Mode 1 instead.
2. **Diff analysis** — run `git diff <last-mapped-sha>..HEAD --name-status` for committed changes, then `git diff HEAD --name-status` for uncommitted changes. Also run `git ls-files --others --exclude-standard` to capture untracked files (treat these as status `A`). Merge all three result sets. If a file appears in both the committed and uncommitted diffs, use the status from the uncommitted diff (`git diff HEAD`) since it reflects the current working tree state. Exclude `ARCHITECTURE.md` from the diff results to avoid self-referential update loops. Check the status prefix: A=added, D=deleted, R=renamed trigger Structure and deps and impact; M=modified requires content inspection.
3. **Scope assessment** — determine which sections are affected by the changes:
   - New/deleted/moved files -> update `structure`
   - Changed imports or new dependencies -> update `deps` (also triggers `impact` update, since import changes affect fan-in counts)
   - New patterns or config changes -> update `conventions`
   - Changes to high-coupling files -> update `impact`
   - If the `impact` section is missing or empty, always regenerate it. (Missing means the `## Change Impact Map` heading is absent; empty means the High-Coupling Zones table has zero data rows.)
4. **Targeted update** — only re-analyze and rewrite affected sections. If multiple sections need updating, launch the corresponding agents in parallel (same as Mode 1 but only for affected sections). If only one section needs updating, run the corresponding agent directly. Impact analysis always runs after the dependency-mapper completes (never in parallel with it).
5. **Preserve** — keep unchanged sections and all `<!-- manual -->` blocks intact
6. **Update metadata** — set `last-mapped` to current HEAD SHA
7. **Summary** — print what sections were updated and why:
   ```
   Updated ARCHITECTURE.md (2 of 4 sections)
     deps: 3 files with changed imports
     impact: fan-in recalculated
     Unchanged: structure, conventions
   ```

### Mode 3: Full Regeneration (`--full`)

Same as Mode 1, but:
1. Read existing `ARCHITECTURE.md` first
2. Extract all `<!-- manual -->` blocks
3. Regenerate everything from scratch
4. Re-inject `<!-- manual -->` blocks using these rules:
   - Each manual block is re-injected after the `##` heading that immediately preceded it in the original file.
   - Manual blocks before the first heading go after the metadata block.
   - Multiple manual blocks under the same heading are preserved in order.
   - If the associated heading no longer exists, append the block at the end of the document under `## Archived Notes`. This is an optional section created only when needed. It is not managed by agents, not listed in `sections-updated`, and is preserved as-is during incremental updates. On subsequent Mode 3 runs, its manual blocks are re-evaluated against current headings.
5. Update metadata

### Mode 4: Section Update (`--section <name>`)

1. Read existing `ARCHITECTURE.md`. If it does not exist, fall back to Mode 1 (full generation) and inform the user: "No existing ARCHITECTURE.md found. Running full generation instead."
2. Re-analyze only the specified section. Note: `--section impact` first checks if the existing ARCHITECTURE.md has a Dependencies section with a Mermaid graph. If yes, it parses the existing graph for fan-in data. If no, it first runs the dependency-mapper agent, then computes the Impact Map.
3. Replace that section, preserve everything else
4. Update metadata
5. Summary:
   ```
   Updated ARCHITECTURE.md (1 of 4 sections)
     conventions: regenerated from current configs
   ```

---

## Agent Prompts

### Structure Mapper Agent

```
You are a project structure analyst. Your job is to understand and document this project's layout.

1. Check for manifest files: package.json, Cargo.toml, pyproject.toml, go.mod, Gemfile, pom.xml, etc.
2. First run `ls` on the project root to discover the actual directory layout. Then glob for source files in discovered directories. Common patterns include `src/`, `lib/`, `app/`, `pkg/`, `cmd/`, `internal/`, or top-level package directories. Adapt to what exists.
3. Glob for test files: tests/**/*.*, test/**/*.*, spec/**/*.*, __tests__/**/*.*
4. Glob for config files: *.config.*, .eslintrc.*, tsconfig.*, etc.
5. Read the manifest to identify language, framework, and project purpose
6. Read 2-3 representative source files to understand the code style

Produce a Structure section with:
- Brief project description (1-2 sentences)
- Annotated directory tree (only directories and key files, not every file)
- Key entry points (application start, CLI, test runner)

Output ONLY the markdown for the Structure section. No preamble.
```

### Dependency Mapper Agent

```
You are a dependency analyst. Your job is to map how modules connect in this project.

1. Grep for import/require/use/include statements across all source files
2. Build a list of: [source_file, imported_module] pairs
3. Count fan-in for each module (how many files import it)
4. Identify the top 10 most-imported modules (these are key interfaces)
5. Read the manifest for external dependencies
6. Grep to find where each external dep is actually used

Produce a Dependencies section with:
- Mermaid dependency graph (show only modules with fan-in >= 2, group by directory). If there are more than 30 modules with fan-in >= 2, show only the top 20 by fan-in count and note the total count of omitted modules.
- External dependencies table (package, purpose, used by)
- Key interfaces list (high fan-in modules with 1-line descriptions)

In addition to the markdown, output a structured JSON data block at the end in the following format (the orchestrator uses this for Impact Analysis):

\`\`\`json
{
  "edges": [
    {"source": "src/api/routes.ts", "target": "src/services/auth.ts"},
    ...
  ],
  "fan_in": {
    "src/models/user.ts": 12,
    ...
  }
}
\`\`\`

Output the markdown for the Dependencies section first, then the JSON block. No other preamble.
```

The orchestrator extracts the last ` ```json ` block from the dependency-mapper's response as the structured data. Everything before it is the Dependencies section markdown.

### Convention Scanner Agent

```
You are a codebase convention analyst. Your job is to identify patterns and practices in this project.

1. Read 3-5 source files from different areas of the codebase
2. Read any linter/formatter configs (.eslintrc, .prettierrc, rustfmt.toml, etc.)
3. Read the language-specific config (tsconfig.json, pyproject.toml, Cargo.toml, go.mod, Gemfile, pom.xml, build.gradle, etc.) if present
4. Check for infrastructure/build configs: Makefile, Dockerfile*, docker-compose*, .env*
5. Read CI config if present (.github/workflows, Jenkinsfile, etc.)
6. Check for a CLAUDE.md, CONTRIBUTING.md, or similar convention docs
7. Look for patterns: error handling, naming, file organization, testing approach

Produce a Conventions section with:
- Patterns subsection (3-7 bullet points of discovered patterns)
- Configuration subsection (key config choices)
- Established Practices subsection (implicit rules observed across multiple files)

Output ONLY the markdown for the Conventions section. No preamble.
```

### Impact Analysis (orchestrator, not a separate agent)

After the three agents complete, the orchestrator:
1. Takes the dependency data from the dependency-mapper
2. Sorts modules by fan-in count
3. Files with fan-in >= 3 -> High-Coupling Zones table
4. Traces 2-3 critical paths (auth, data, API) by following import chains
5. Identifies leaf nodes (fan-out only, fan-in = 0) -> Safe Zones
6. Writes the Change Impact Map section

---

## Diff-Aware Scoping Logic

When running an incremental update, the orchestrator determines which sections need updating:

```
Changed files from git diff --name-status:
  -> Any new/deleted/renamed files (A/D/R)?  YES -> update structure. R also triggers deps and impact (renames change import paths).
  -> Any changed import statements?          YES -> update deps (also triggers impact update)
  -> Any changed config files?               YES -> update conventions
  -> Any changed files in high-coupling list? YES -> update impact
  -> Impact section missing or empty?        YES -> update impact
  -> None of the above?                      -> Skip update, print "No structural changes since last map"

Config files are: `*.config.*`, `.eslintrc*`, `.prettierrc*`, `rustfmt.toml`, `tsconfig*`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `Gemfile`, `pom.xml`, `build.gradle*`, `.env*`, `Makefile`, `Dockerfile*`, `docker-compose*`, `.github/workflows/*`, `Jenkinsfile`.
```

To detect changed imports without re-reading every file:
```bash
# Committed changes
git diff <last-sha>..HEAD -G "^(import |import\(|from ['\"]|require\(|require |use |use;|pub use |#include|#import|@import)" --name-only
# Uncommitted changes
git diff HEAD -G "^(import |import\(|from ['\"]|require\(|require |use |use;|pub use |#include|#import|@import)" --name-only
```

This is a heuristic and may need per-language tuning. The `from ['\"]` pattern reduces false positives from prose by requiring a quote character. Indented imports (e.g., dynamic imports inside functions) are still caught because the file appears in the broader `--name-status` diff. False positives are acceptable since they only cause unnecessary but harmless section updates. Untracked files (from `git ls-files --others`) use file-level status only — their imports are not inspected until they are committed or staged. This gives only files where import lines changed, which is the trigger for dependency updates.

---

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| Monorepo with multiple packages | Discover sub-packages by looking for nested manifest files (package.json, Cargo.toml, go.mod, etc.). Each sub-package gets a subsection under Structure with its own directory tree. The dependency graph includes cross-package edges. Limit to 2 levels of nesting. |
| No git history (fresh repo) | If `git rev-parse HEAD` fails (no commits), run in full mode and set `last-mapped` to `0000000`. The next incremental run treats this as "full remap needed." |
| `ARCHITECTURE.md` exists but has no metadata block | Parse the existing file for known section headings (`## Structure`, `## Dependencies`, `## Conventions`, `## Change Impact Map`). Content under recognized headings is treated as seed data for those sections (read for context, then regenerated). Content not under recognized headings is wrapped in `<!-- manual -->` blocks. Warn the user about what content was wrapped and that they can remove the `<!-- manual -->` tags afterward. Add the metadata block. Then run Mode 3 (full regeneration) so the manual-wrapped content is preserved via Mode 3's re-injection logic. |
| Very large repo (>1000 files) | Limit directory tree to 2 levels deep. Focus dependency graph on top 20 fan-in modules. |
| Binary-heavy repo | Skip binary files in analysis. Note binary directories in Structure with counts only. |
| No source files (docs-only repo) | Generate Structure and Conventions sections only. Conventions should note that no source files were found. Skip Dependencies and Change Impact Map. Set `sections-updated` to `structure,conventions`. |

---

## Error Handling

| Condition | Behavior |
|-----------|----------|
| Not a git repo | Run in full mode without diff-awareness. Warn the user: "Not a git repository. Running full generation without diff-awareness." |
| Git unavailable | Run in full mode without diff-awareness. Warn the user: "Git not found. Running full generation without diff-awareness." |
| Write permission denied | Print error: "Cannot write to ARCHITECTURE.md — permission denied." Exit without modifying any files. |
| Agent failure | Retry the failed agent once. If it fails again, skip that section and warn the user: "Could not generate <section>. Skipping." |
| Corrupt metadata block | Treat as no metadata — regenerate all sections (same as Mode 1). Warn the user: "Metadata block is corrupt. Running full generation." |
| ARCHITECTURE.md read-only | Print error: "ARCHITECTURE.md is read-only. Cannot update." Exit without modifying any files. |

---

## Integration Points

### Consumed by `/adversarial-review`

The adversarial review reads `ARCHITECTURE.md` to:
- Understand which modules are high-risk (Impact Map)
- Scope diff-based reviews to affected areas
- Cross-reference spec changes against the dependency graph
- Know project conventions when evaluating code

### Consumed by `/plan`

Planning skills read `ARCHITECTURE.md` to:
- Understand where new code should live (Structure)
- Know which modules a new feature will touch (Dependencies)
- Follow existing conventions (Conventions)
- Identify blast radius of proposed changes (Impact Map)

### Consumed by new sessions

When Claude starts a new session, `ARCHITECTURE.md` provides instant project context without re-discovering everything from scratch.

---

## Success Criteria

After running `/map`, the following should be true:
1. `ARCHITECTURE.md` exists at the repo root
2. All four sections are populated with accurate information
3. The metadata block contains the current HEAD SHA
4. Running `/map` again immediately produces "No structural changes since last map"
5. After making a code change and committing, running `/map` updates only affected sections
6. `<!-- manual -->` blocks survive both incremental and full updates
7. The Mermaid diagram renders correctly on GitHub

---

## Updated `/adversarial-review` Integration

> The following describes the integration contract between `/map` and `/adversarial-review`. Full specification of these changes to `/adversarial-review` is out of scope for this document and should be specified in a separate spec.

When `/adversarial-review` runs, it should:

1. Check if `ARCHITECTURE.md` exists
   - If yes: read it and include in reviewer context
   - If no: suggest running `/map` first, or run a lightweight discovery inline
2. Read the following from `ARCHITECTURE.md` for review context:
   - **Impact Map** — identify which changed files are in high-coupling zones to flag high-risk changes
   - **Dependencies** — cross-reference the Mermaid graph to understand blast radius of changes
   - **Conventions** — evaluate code against established project conventions
3. Read project-level overrides from `.claude/review-config.md` if it exists (custom reviewer focus areas, files to ignore, severity overrides)

---

## File Locations

| File | Location | Committed to git? |
|------|----------|-------------------|
| `/map` command | `~/.claude/commands/map.md` | No (user-level) |
| `/adversarial-review` command | `~/.claude/commands/adversarial-review.md` | No (user-level) |
| `ARCHITECTURE.md` artifact | `<repo-root>/ARCHITECTURE.md` | Yes |
| Review config overrides | `<repo-root>/.claude/review-config.md` | Yes (optional) |

---

## Command File: `map.md`

The full content of `~/.claude/commands/map.md`:

````markdown
# /map — Generate or update ARCHITECTURE.md

You are the `/map` orchestrator. Your job is to generate or update an `ARCHITECTURE.md` file at the root of this repository.

## Step 1: Parse arguments

Split `$ARGUMENTS` on whitespace.

- `--full` -> Mode 3 (full regeneration)
- `--section <name>` -> Mode 4 (section update). Valid names: `structure`, `deps`, `conventions`, `impact`. If the name is unrecognized, print the valid options and stop.
- `--full` and `--section` both present -> `--full` takes precedence.
- Unknown flags -> ignore with a warning.
- No arguments -> continue to mode selection below.

## Step 2: Select mode

- If `--full` was passed: **Mode 3** (full regeneration).
- If `--section` was passed: **Mode 4** (section update). If `ARCHITECTURE.md` does not exist, fall back to Mode 1 and inform the user.
- If `ARCHITECTURE.md` does not exist: **Mode 1** (first run).
- If `ARCHITECTURE.md` exists with valid metadata: **Mode 2** (incremental update).
- If `ARCHITECTURE.md` exists without metadata: parse for known headings, wrap unrecognized content as `<!-- manual -->`, warn the user about what was wrapped, then run **Mode 3** (full regeneration) so manual-wrapped content is preserved via re-injection.

## Step 3: Execute

### Mode 1 / Mode 3: Full generation

1. Read project files to discover language, framework, and structure.
2. Dispatch three agents in parallel using the Agent tool (subagent_type: "general-purpose"):
   - **structure-mapper**: Produces `## Structure` markdown. Agent prompt:
     > You are a project structure analyst. Your job is to understand and document this project's layout.
     > 1. Check for manifest files: package.json, Cargo.toml, pyproject.toml, go.mod, Gemfile, pom.xml, etc.
     > 2. First run `ls` on the project root to discover the actual directory layout. Then glob for source files in discovered directories. Common patterns include `src/`, `lib/`, `app/`, `pkg/`, `cmd/`, `internal/`, or top-level package directories. Adapt to what exists.
     > 3. Glob for test files: tests/**/*.*, test/**/*.*, spec/**/*.*, __tests__/**/*.*
     > 4. Glob for config files: *.config.*, .eslintrc.*, tsconfig.*, etc.
     > 5. Read the manifest to identify language, framework, and project purpose
     > 6. Read 2-3 representative source files to understand the code style
     > Produce a Structure section with: brief project description (1-2 sentences), annotated directory tree (only directories and key files, not every file), key entry points (application start, CLI, test runner).
     > Output ONLY the markdown for the Structure section. No preamble.
   - **dependency-mapper**: Produces `## Dependencies` markdown + structured JSON. Agent prompt:
     > You are a dependency analyst. Your job is to map how modules connect in this project.
     > 1. Grep for import/require/use/include statements across all source files
     > 2. Build a list of: [source_file, imported_module] pairs
     > 3. Count fan-in for each module (how many files import it)
     > 4. Identify the top 10 most-imported modules (these are key interfaces)
     > 5. Read the manifest for external dependencies
     > 6. Grep to find where each external dep is actually used
     > Produce a Dependencies section with: Mermaid dependency graph (show only modules with fan-in >= 2, group by directory; if more than 30, show top 20 and note omitted count), external dependencies table (package, purpose, used by), key interfaces list (high fan-in modules with 1-line descriptions).
     > In addition to the markdown, output a structured JSON data block at the end: `{"edges": [{"source": "...", "target": "..."}], "fan_in": {"file": count}}`.
     > Output the markdown for the Dependencies section first, then the JSON block. No other preamble.
     Extract the last ```json block from the dependency-mapper's response as the structured data. Everything before it is the Dependencies section markdown.
   - **convention-scanner**: Produces `## Conventions` markdown. Agent prompt:
     > You are a codebase convention analyst. Your job is to identify patterns and practices in this project.
     > 1. Read 3-5 source files from different areas of the codebase
     > 2. Read any linter/formatter configs (.eslintrc, .prettierrc, rustfmt.toml, etc.)
     > 3. Read the language-specific config (tsconfig.json, pyproject.toml, Cargo.toml, go.mod, Gemfile, pom.xml, build.gradle, etc.) if present
     > 4. Check for infrastructure/build configs: Makefile, Dockerfile*, docker-compose*, .env*
     > 5. Read CI config if present (.github/workflows, Jenkinsfile, etc.)
     > 6. Check for a CLAUDE.md, CONTRIBUTING.md, or similar convention docs
     > 7. Look for patterns: error handling, naming, file organization, testing approach
     > Produce a Conventions section with: Patterns subsection (3-7 bullet points), Configuration subsection (key config choices), Established Practices subsection (implicit rules observed across multiple files).
     > Output ONLY the markdown for the Conventions section. No preamble.
   If an agent fails, retry once. If it fails again, skip that section and warn the user.
3. After dependency-mapper completes, compute the Impact Analysis from its JSON data:
   - Sort modules by fan-in. Files with fan-in >= 3 go in High-Coupling Zones.
   - Trace 2-3 critical paths. Identify leaf nodes for Safe Zones.
4. Assemble all sections into `ARCHITECTURE.md` with the metadata block.
5. For Mode 3 only: re-inject extracted `<!-- manual -->` blocks per the re-injection rules.
6. Print a summary of what was generated.

### Mode 2: Incremental update

1. Parse `ARCHITECTURE.md` and extract the metadata block. If `last-mapped` is `0000000`, run Mode 1 instead.
2. Run `git diff <last-mapped-sha>..HEAD --name-status` and `git diff HEAD --name-status`. Also run `git ls-files --others --exclude-standard` to capture untracked files (treat as status `A`). Merge all results. If a file appears in both diffs, use the uncommitted status. Exclude `ARCHITECTURE.md`.
3. Determine affected sections using the scoping logic (see Diff-Aware Scoping Logic in spec).
4. If no sections affected, print "No structural changes since last map" and stop.
5. Launch agents for affected sections (parallel if multiple, direct if one). Impact analysis always runs after dependency-mapper.
6. Replace only affected sections. Preserve unchanged sections and `<!-- manual -->` blocks.
7. Update metadata with current HEAD SHA.
8. Print a summary of what was updated.

### Mode 4: Section update

1. Parse `ARCHITECTURE.md`.
2. For `--section impact`: check if Dependencies section has a Mermaid graph. If not, run dependency-mapper first.
3. Run the corresponding agent for the requested section.
4. Replace that section. Preserve everything else.
5. Update metadata.
6. Print a summary.

## Error handling

- Not a git repo or git unavailable: run in full mode without diff-awareness, warn the user.
- Write permission denied or read-only file: print error and stop.
- Agent failure: retry once, then skip with warning.
- Corrupt metadata: treat as no metadata, run full generation.
````
