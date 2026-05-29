# Team Development Standards

A comprehensive, opinionated guide for building Laravel applications across all team projects. These rules are **strict, must-follow standards** — deviations require justification.

## Idea

Build a single source of truth for how the team writes Laravel. The goal is to eliminate stylistic debates in PRs, ensure new developers onboard fast, and keep code shape consistent across projects so any team member can drop into any codebase.

## Dependencies (Standard Stack)

Every Laravel project adopts this baseline:

### Core packages
- `spatie/laravel-data` — used as DTO transport layer (typed `::from()` hydration). Validation attributes NOT used (validation lives in FormRequests).
- `spatie/laravel-model-states` — state machines for entities with non-trivial status flows

### Code quality (CI-enforced)
- `laravel/pint` — formatter (Laravel preset)
- `larastan/larastan` — static analysis (level 5, ratchet up)
- `rector/rector` — automated refactoring (`--dry-run` in CI)
- `barryvdh/laravel-ide-helper` — IDE autocomplete (dev only)

### Testing
- `pestphp/pest` — test framework
- `pestphp/pest-plugin-laravel` — Laravel-specific helpers
- Postgres (test database matches production engine)
- Laravel's built-in fakes (`Http::fake()`, `Event::fake()`, `Queue::fake()`, etc.)
- Custom `Fake*` Service classes (in `tests/Fakes/`) for vendor-SDK Services

## Format

- This is the **content-first** version of the guide.
- Final delivery format (Markdown repo / Notion / PDF) decided later.

## Contents (Sections)

Sections are ordered by **technical priority**, not numerically. Earlier sections influence later ones; the build order respects dependencies.

| Build order | # | Section | Status |
|---|---|---|---|
| 1 | 3 | [Code style & static analysis](./03-code-style-static-analysis.md) | ✅ Locked |
| 2 | 4 | [Architecture patterns](./04-architecture-patterns.md) | ✅ Locked |
| 3 | 9 | [Testing strategy](./09-testing-strategy.md) | ✅ Locked |
| 4 | 12 | [Git workflow & code review](./12-git-workflow.md) | ✅ Locked |
| 5 | 5 | [Database & Eloquent](./05-database-eloquent.md) | ⏳ Pending |
| 6 | 11 | [Security practices](./11-security.md) | ⏳ Pending |
| 7 | 7 | [Authentication & authorization](./07-auth.md) | ⏳ Pending |
| 8 | 10 | [Logging, monitoring & error tracking](./10-logging-monitoring.md) | ⏳ Pending |
| 9 | 8 | [Background jobs in production](./08-background-jobs.md) | ⏳ Pending |
| 10 | 13 | [CI/CD pipeline](./13-ci-cd.md) | ⏳ Pending |
| 11 | 14 | [Deployment & infrastructure](./14-deployment.md) | ⏳ Pending |
| 12 | 6 | [API & frontend integration](./06-api-frontend.md) | ⏳ Pending |
| 13 | 15 | [Performance & caching](./15-performance-caching.md) | ⏳ Pending |
| 14 | 1 | [Local development environment](./01-local-dev.md) | ⏳ Pending |
| 15 | 2 | [Project scaffolding](./02-scaffolding.md) | ⏳ Pending |
| 16 | 16 | [Documentation & onboarding](./16-documentation.md) | ⏳ Pending |

## Architecture Reference

The [`architecture/`](./architecture/) folder contains a deep reference for each architectural building block. Start with [`architecture/README.md`](./architecture/README.md).

A printable poster of the architecture is available at [`architecture-map.html`](./architecture-map.html).

## Progress Tracker

**Locked:** 4 sections (3, 4, 9, 12).
**Remaining:** 12 sections.

**Next up:** Section 5 (Database & Eloquent).

## Conventions for this Guide

- **Locked rules** are stated as imperatives ("Use X", "Never do Y").
- **Rationale** is included where it helps; omitted where the rule is obvious.
- **Examples** are language-generic when possible, Laravel-specific when needed. Not tied to any specific product or domain.
- **Strictness:** strict by default. Every "exception" must be documented at the call site with a comment.
