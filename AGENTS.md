# Agent Instructions

This project uses strict team development standards. Any AI coding assistant (OpenCode, Sourcegraph Cody, Cline, Continue, Aider, and other tools that read `AGENTS.md`) **must read the skills in `.ai/skills/` before generating code.**

## Skills to load

Load these files in this order at the start of a session:

1. **`.ai/skills/architecture.md`** — folder structure, class naming, FormRequest + DTO + Resource triad, Action/Query/Service layering, `final` rule, dependency direction, exception handling, model lifecycle, state machines, forbidden patterns
2. **`.ai/skills/testing.md`** — Pest conventions, Postgres + RefreshDatabase, feature vs unit boundaries, `Http::fake()` vs interface fakes, factory patterns, what NOT to test

## Rules of behavior

- Every file you generate or modify must conform to the loaded skills
- When in doubt, prefer the more explicit, type-safe, testable option
- Show complete files with `<?php`, `declare(strict_types=1);`, full namespace, all `use` statements — no partial snippets unless explicitly asked
- All concrete classes are `final` unless abstract
- All class names carry their type suffix: `PlaceOrderAction`, `OrderPlacedEvent`, `OrderResource`, `CreateOrderData`
- The forbidden-patterns list in `architecture.md` is non-negotiable

## Source-of-truth precedence

The skills in `.ai/skills/` are the source of truth — not the existing code in the repo. If you detect drift between code and skills, flag it; do not silently match existing code's style.

## Human-readable references

For rationale and discussion (not for code generation):

- `README.md` — overview
- `04-architecture-patterns.md` — architecture spec
- `09-testing-strategy.md` — testing spec
- `12-git-workflow.md` — Git workflow
- `architecture/` — per-component deep reference
