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

---

## Output: `ARCHITECTURE.md` Format

The generated document has four sections plus metadata. Each section is scaled to the project's complexity — a 5-file project gets a few lines per section, a 500-file monorepo gets detailed subsections.

### Metadata Block

Hidden HTML comment at the top of the file, used for diff-aware updates:

```markdown
<!-- map-metadata
last-mapped: abc1234
last-mapped-date: 2026-03-12
sections-updated: structure,deps,conventions,impact
-->
```

Fields:
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

### External Dependencies

| Package | Purpose | Used by |
|---------|---------|---------|
| express | HTTP server | src/api/* |
| prisma | Database ORM | src/models/* |
| zod | Validation | src/utils/validation.ts, src/api/* |

### Key Interfaces

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

**Discovery approach**: Count import fan-in for every file. Files with fan-in > 3 go in high-coupling. Trace call chains for critical user flows. Leaf nodes (fan-in = 0, fan-out only) are safe zones.

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
2. **Parallel analysis** — launch 3 agents simultaneously:
   - `structure-mapper` — globs, tree, file purposes, entry points
   - `dependency-mapper` — grep imports, build dep graph, identify key interfaces
   - `convention-scanner` — read representative files, identify patterns, check configs
3. **Impact analysis** — after agents complete, the orchestrator computes fan-in counts and critical paths from the dependency data
4. **Write** — assemble all sections into `ARCHITECTURE.md` with metadata block
5. **Summary** — print what was generated for the user

### Mode 2: Incremental Update (ARCHITECTURE.md exists, no `--full`)

1. **Read existing** — parse current `ARCHITECTURE.md`, extract metadata block
2. **Diff analysis** — run `git diff <last-mapped-sha>..HEAD --name-only` to get changed files
3. **Scope assessment** — determine which sections are affected by the changes:
   - New/deleted/moved files -> update Structure
   - Changed imports or new dependencies -> update Dependencies
   - New patterns or config changes -> update Conventions
   - Changes to high-coupling files -> update Impact Map
4. **Targeted update** — only re-analyze and rewrite affected sections
5. **Preserve** — keep unchanged sections and all `<!-- manual -->` blocks intact
6. **Update metadata** — set `last-mapped` to current HEAD SHA
7. **Summary** — print what sections were updated and why

### Mode 3: Full Regeneration (`--full`)

Same as Mode 1, but:
1. Read existing `ARCHITECTURE.md` first
2. Extract all `<!-- manual -->` blocks
3. Regenerate everything from scratch
4. Re-inject `<!-- manual -->` blocks in their original positions (matched by nearest section heading)
5. Update metadata

### Mode 4: Section Update (`--section <name>`)

1. Read existing `ARCHITECTURE.md`
2. Re-analyze only the specified section
3. Replace that section, preserve everything else
4. Update metadata

---

## Agent Prompts

### Structure Mapper Agent

```
You are a project structure analyst. Your job is to understand and document this project's layout.

1. Check for manifest files: package.json, Cargo.toml, pyproject.toml, go.mod, Gemfile, pom.xml, etc.
2. Glob for source files: src/**/*.*, lib/**/*.*, app/**/*.*, etc.
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
- Mermaid dependency graph (show only modules with fan-in >= 2, group by directory)
- External dependencies table (package, purpose, used by)
- Key interfaces list (high fan-in modules with 1-line descriptions)

Output ONLY the markdown for the Dependencies section. No preamble.
```

### Convention Scanner Agent

```
You are a codebase convention analyst. Your job is to identify patterns and practices in this project.

1. Read 3-5 source files from different areas of the codebase
2. Read any linter/formatter configs (.eslintrc, .prettierrc, rustfmt.toml, etc.)
3. Read the TypeScript/build/compiler config if present
4. Read CI config if present (.github/workflows, Jenkinsfile, etc.)
5. Check for a CLAUDE.md, CONTRIBUTING.md, or similar convention docs
6. Look for patterns: error handling, naming, file organization, testing approach

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
Changed files from git diff:
  -> Any new/deleted/renamed files?          YES -> update Structure
  -> Any changed import statements?          YES -> update Dependencies
  -> Any changed config files?               YES -> update Conventions
  -> Any changed files in high-coupling list? YES -> update Impact Map
  -> None of the above?                      -> Skip update, print "No structural changes since last map"
```

To detect changed imports without re-reading every file:
```bash
git diff <last-sha>..HEAD -G "^(import|from|require|use)" --name-only
```

This gives only files where import lines changed, which is the trigger for dependency updates.

---

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| Monorepo with multiple packages | Map the root, note sub-packages in Structure. Each package gets its own subsection. |
| No git history (fresh repo) | Run in full mode, set `last-mapped` to current HEAD (even if it's the initial commit) |
| `ARCHITECTURE.md` exists but has no metadata block | Treat as manual file. Read it for context, regenerate with metadata, preserve all existing content as `<!-- manual -->` |
| Very large repo (>1000 files) | Limit directory tree to 2 levels deep. Focus dependency graph on top 20 fan-in modules. |
| Binary-heavy repo | Skip binary files in analysis. Note binary directories in Structure with counts only. |
| No source files (docs-only repo) | Generate Structure section only. Skip Dependencies and Impact Map. Note in Conventions. |

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

When `/adversarial-review` runs, it should:

1. Check if `ARCHITECTURE.md` exists
   - If yes: read it and include in reviewer context
   - If no: suggest running `/map` first, or run a lightweight discovery inline
2. Support `$ARGUMENTS` for diff-scoped review:
   - `/adversarial-review` (no args) — full review (current behavior)
   - `/adversarial-review --diff` — review only changes since last commit
   - `/adversarial-review --diff <sha>` — review only changes since specified commit
3. When diff-scoped, cross-reference changed files against the Impact Map to flag high-risk changes
4. Read project-level overrides from `.claude/review-config.md` if it exists (custom reviewer focus areas, files to ignore, severity overrides)

---

## File Locations

| File | Location | Committed to git? |
|------|----------|-------------------|
| `/map` command | `~/.claude/commands/map.md` | No (user-level) |
| `/adversarial-review` command | `~/.claude/commands/adversarial-review.md` | No (user-level) |
| `ARCHITECTURE.md` artifact | `<repo-root>/ARCHITECTURE.md` | Yes |
| Review config overrides | `<repo-root>/.claude/review-config.md` | Yes (optional) |
