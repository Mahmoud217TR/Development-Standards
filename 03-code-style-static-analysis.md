# Section 3: Code Style & Static Analysis

## Goal

Eliminate style debates in PRs. Catch type errors, dead code, and outdated patterns before they reach review. Make code shape predictable across the team.

## Tools

| Tool | Purpose | Where it runs |
|---|---|---|
| **Pint** | Formatter (Laravel preset) | CI |
| **Larastan** | Static analysis (PHPStan + Laravel rules) | CI |
| **Rector** | Automated refactoring (advisory) | CI (`--dry-run`) |
| **IDE Helper** | Autocomplete stubs (dev) | Local only |

## Configuration

### Pint
- Preset: **Laravel** (default)
- No custom rules unless team agrees on an addition
- Config file: `pint.json` at project root (commit it)

### Larastan
- Level: **5** initially. Can only be raised, never lowered.
- Bumping a level requires a dedicated PR that fixes all violations — no mixing with feature work.
- Config file: `phpstan.neon` at project root

### Rector
- Initial setup: run once on a clean branch, commit the diff as `chore: initial rector baseline`.
- Rule sets (conservative start):
  - PHP version rules (matching project's PHP version)
  - Laravel upgrade rules
  - Dead code removal
  - `FinalizeClassesWithoutChildrenRector` (enforces our `final` rule from Section 4)
- Reviewed and expanded quarterly.
- Config file: `rector.php` at project root

### IDE Helper
- Required dev dependency
- Generate after model changes: `php artisan ide-helper:models -W`
- `.gitignore` the generated `_ide_helper*.php` files

## CI Behavior

All three tools run on every PR. PR cannot merge until they pass.

| Tool | Command | Fails PR if |
|---|---|---|
| Pint | `vendor/bin/pint --test` | Any formatting drift |
| Larastan | `vendor/bin/phpstan analyse` | Any level-5 violation |
| Rector | `vendor/bin/rector process --dry-run` | Any refactoring opportunity found |

## Developer Workflow

1. Write code locally.
2. Before pushing, run all three locally:
   ```bash
   vendor/bin/pint
   vendor/bin/phpstan analyse
   vendor/bin/rector process
   ```
3. Push → CI verifies on full codebase.
4. PR blocked until all green.

## Rules

- **No pre-commit hooks.** CI is the single source of enforcement. Avoids bypass concerns and keeps tooling simple.
- **No inline disabling without justification.** `// phpcs:ignore`, `// @phpstan-ignore-next-line`, etc. require a comment explaining why.
- **Larastan level is a ratchet** — only goes up, never down.
- **New projects** start with an initial Rector pass committed before any feature work.

## Why these choices

- **Pint over PHP-CS-Fixer directly:** Laravel preset is curated; zero config.
- **Larastan level 5 over higher:** catches real bugs without forcing massive type-annotation work on day one. Ratchet later.
- **Rector advisory (dry-run) over auto-apply:** humans review refactor diffs before they ship. No surprise rewrites.
- **CI only over hooks:** developers can bypass hooks with `--no-verify`. CI cannot be bypassed. One source of truth.
