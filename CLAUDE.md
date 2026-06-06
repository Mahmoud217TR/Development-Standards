# Claude Code Instructions

This project follows strict team development standards. **You must read and follow the skills in `.ai/skills/` before generating any code.**

## Skills to load

Read these files at the start of every session, in this order:

1. **`.ai/skills/architecture.md`** — class naming, folder structure, the FormRequest + DTO + Resource triad, Actions/Queries/Services layering, `final` rule, dependency direction, exception handling, model lifecycle, state machines, and the forbidden-patterns list
2. **`.ai/skills/testing.md`** — Pest conventions, Postgres + RefreshDatabase, feature vs unit test boundaries, `Http::fake()` vs interface fakes, factory patterns, what NOT to test

## Behavior

- Every file you create or modify must conform to these rules
- When in doubt, prefer the more explicit, type-safe, testable option
- Always show complete files (with `<?php`, `declare(strict_types=1);`, namespace, all `use` statements) — never partial snippets unless explicitly asked
- All concrete classes are `final` unless they are abstract bases
- All class names carry their type suffix: `PlaceOrderAction`, `OrderPlacedEvent`, `OrderResource`, `CreateOrderData`, etc.
- The forbidden-patterns list in `architecture.md` is non-negotiable — refactor immediately if you detect them

## Human-readable standards

For the full standards with rationale and discussion, see:

- `README.md` — overview and progress tracker
- `04-architecture-patterns.md` — architecture rules
- `09-testing-strategy.md` — testing rules
- `12-git-workflow.md` — Git and PR rules
- `architecture/` — per-component deep reference
- `architecture-map.html` — visual map of the request flow

## Reporting drift

If you detect that code in this repo violates the standards in `.ai/skills/`, flag it explicitly in your response — do not silently match the existing code's style. The skills file is the source of truth, not the existing code.
