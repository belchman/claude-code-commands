# claude-code-commands

A collection of slash commands for [Claude Code](https://claude.ai/claude-code).

## Commands

### `/map`
Generates or updates an `ARCHITECTURE.md` file at the root of your repository — a living map of your project's structure, dependencies, conventions, and high-coupling zones.

- Works on any language or framework (polyglot-adaptive)
- Dispatches parallel agents for structure, dependencies, and conventions
- Incremental updates via `git diff` — only re-maps what changed
- Flags high-risk zones by fan-in analysis

**Usage:**
```
/map               # First run or incremental update
/map --full        # Force full regeneration
/map --section deps  # Update one section (structure | deps | conventions | impact)
```

---

### `/adversarial-review`
Runs three parallel reviewer agents against your project's docs, config, and tests — finding contradictions, gaps, and missing pieces, then fixing what you approve.

- **Spec reviewer** — contradictions, missing definitions, ambiguous behavior
- **Config reviewer** — broken hooks, permission gaps, env var issues
- **Coverage reviewer** — untested exports, unimplemented contracts, framework mismatches
- Uses `ARCHITECTURE.md` (from `/map`) for blast-radius analysis when available
- `--diff` mode scopes the review to changes since the last map

**Usage:**
```
/adversarial-review              # Full review of all project files
/adversarial-review --diff       # Review only what changed since last /map
/adversarial-review path/to/file # Review a specific file or glob
```

---

## Install

```bash
git clone https://github.com/belchman/claude-code-commands
claude plugins install ./claude-code-commands
```

## Recommended workflow

1. Run `/map` on a new project to generate `ARCHITECTURE.md`
2. Run `/adversarial-review` to find gaps in your docs and config
3. After changes, run `/map --section deps` or `/adversarial-review --diff` to stay current

## Design

See [`docs/specs/`](docs/specs/) for detailed specifications behind each command.

## License

MIT — [Matthew Belchak](https://github.com/belchman)
