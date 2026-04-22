# claude-code-commands

A marketplace of individually-installable commands and skills for [Claude Code](https://claude.ai/claude-code). Each plugin stands alone — install only what you need.

## Install

Add the marketplace once:

```
/plugin marketplace add belchman/claude-code-commands
```

Then install any subset:

```
/plugin install map@claude-code-commands
/plugin install adversarial-review@claude-code-commands
/plugin install crap@claude-code-commands
```

## Plugins

### `map` — slash command `/map`

Generates or updates an `ARCHITECTURE.md` at the root of your repository: a living map of structure, dependencies, conventions, and high-coupling zones.

- Works on any language or framework (polyglot-adaptive)
- Dispatches parallel agents for structure, dependencies, and conventions
- Incremental updates via `git diff` — only re-maps what changed
- Flags high-risk zones by fan-in analysis

```
/map                  # first run or incremental update
/map --full           # force full regeneration
/map --section deps   # update one section (structure | deps | conventions | impact)
```

---

### `adversarial-review` — slash command `/adversarial-review`

Runs three parallel reviewer agents against your project's docs, config, and tests — finding contradictions, gaps, and missing pieces, then fixing what you approve.

- **Spec reviewer** — contradictions, missing definitions, ambiguous behavior
- **Config reviewer** — broken hooks, permission gaps, env var issues
- **Coverage reviewer** — untested exports, unimplemented contracts, framework mismatches
- Uses `ARCHITECTURE.md` (from `/map`) for blast-radius analysis when available
- `--diff` mode scopes the review to changes since the last map

```
/adversarial-review                 # full review of all project files
/adversarial-review --diff          # review only what changed since last /map
/adversarial-review path/to/file    # review a specific file or glob
```

---

### `crap` — skill (auto-triggers on `/crap` or risky-code questions)

Ranks functions by CRAP score (cyclomatic complexity × lack of real test coverage) on the current branch, then proposes either a refactor or missing tests for the worst offender.

- CRAP = `cc² × (1 − eff_cov)³ + cc`, where `eff_cov = line_cov × mutation_kill_rate`
- Cache + baseline on disk so repeat runs are fast and regressions are gated
- Per-language detectors for coverage/mutation (Python, JS/TS, Go, Rust, Java/Kotlin, C/C++) — see [`plugins/crap/skills/crap/detectors.md`](plugins/crap/skills/crap/detectors.md)
- After ranking, offers a language-aware refactor or a test-case draft for the single worst function

```
/crap                   # changed-files scope, threshold from .crap.yml or 30
/crap 20                # threshold override
/crap --full            # whole tree
/crap --dry-run         # skip refactor/tests step
/crap --set-baseline    # write .crap-baseline.json and exit
```

**Dependencies.** `/crap` needs `python3` and `lizard` (`pip install lizard`). Coverage and mutation tools are language-specific and installed on demand — the skill prints the exact install command when something is missing. See [`plugins/crap/skills/crap/crap.yml.example`](plugins/crap/skills/crap/crap.yml.example) for project configuration.

---

## Recommended workflow

1. `/map` on a new project → generates `ARCHITECTURE.md`
2. `/adversarial-review` → finds gaps in docs and config
3. `/crap` → finds risky, under-tested code to address next
4. After changes, `/map --section deps` or `/adversarial-review --diff` to stay current

## License

MIT — [belchman](https://github.com/belchman)
