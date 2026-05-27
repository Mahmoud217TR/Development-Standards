# Laravel Team Standards

A comprehensive, opinionated guide for building Laravel applications across all team projects. These rules are **strict, must-follow standards** — deviations require justification.

## Idea

Build a single source of truth for how the team writes Laravel. The goal is to eliminate stylistic debates in PRs, ensure new developers onboard fast, and keep code shape consistent across projects so any team member can drop into any codebase.

## Dependencies (Standard Stack)

Every Laravel project adopts this baseline:

### PHP & Framework
- PHP 8.3+
- Laravel 11+

### Core packages
- `spatie/laravel-data` — input validation, output serialization, DTOs, TypeScript generation
- `spatie/laravel-model-states` — state machines for entities with non-trivial status flows
- `spatie/laravel-typescript-transformer` — auto-generate TS types from Data classes

### Code quality (CI-enforced)
- `laravel/pint` — formatter (Laravel preset)
- `larastan/larastan` — static analysis (level 5, ratchet up)
- `rector/rector` — automated refactoring (`--dry-run` in CI)
- `barryvdh/laravel-ide-helper` — IDE autocomplete (dev only)

### Testing
- `pestphp/pest` — test framework
- `pestphp/pest-plugin-laravel` — Laravel-specific helpers
- (additional testing packages TBD in Section 9)

## Format

- This is the **content-first** version of the guide.
- Final delivery format (Markdown repo / Notion / PDF) decided later.

## Contents (Sections)

| # | Section | Status |
|---|---|---|
| 1 | Local development environment | ⏭️ Skipped |
| 2 | Project scaffolding | ⏭️ Skipped |
| 3 | [Code style & static analysis](./03-code-style-static-analysis.md) | ✅ Locked |
| 4 | [Architecture patterns](./04-architecture-patterns.md) | ✅ Locked |
| 5 | Database & Eloquent | ⏭️ Skipped |
| 6 | API & frontend integration | ⏭️ Skipped |
| 7 | Authentication & authorization | ⏭️ Skipped |
| 8 | Background jobs & scheduling | ⏭️ Skipped |
| 9 | [Testing strategy](./09-testing-strategy.md) | 🔄 In progress |
| 10 | [Logging, monitoring & error tracking](./10-logging-monitoring.md) | ⏳ Pending |
| 11 | [Security practices](./11-security.md) | ⏳ Pending |
| 12 | [Git workflow & code review](./12-git-workflow.md) | ⏳ Pending |
| 13 | [CI/CD pipeline](./13-ci-cd.md) | ⏳ Pending |
| 14 | [Deployment & infrastructure](./14-deployment.md) | ⏳ Pending |
| 15 | [Performance & caching](./15-performance-caching.md) | ⏳ Pending |
| 16 | Documentation & onboarding | ⏭️ Skipped |

## Architecture Reference

The [`architecture/`](./architecture/) folder contains a deep reference for each architectural building block — definitions, rules, signatures, and examples. Start with [`architecture/README.md`](./architecture/README.md).

## Progress Tracker

**Current step:** Section 9 (Testing) — 8 open questions awaiting decisions.

**Next sections after 9:** 10 → 11 → 12 → 13 → 14 → 15.

**How to use this tracker:** When a section is locked, its status moves to ✅ and the file is finalized. In-progress sections track open questions inside their own file.

## Conventions for this Guide

- **Locked rules** are stated as imperatives ("Use X", "Never do Y").
- **Rationale** is included where it helps; omitted where the rule is obvious.
- **Examples** are language-generic when possible, Laravel-specific when needed. Not tied to any specific product or domain.
- **Strictness:** strict by default. Every "exception" must be documented at the call site with a comment.
