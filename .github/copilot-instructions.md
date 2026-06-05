# GitHub Copilot Instructions

This project follows strict team development standards. Read the skills in `.ai/skills/` before generating code.

## Skills to load

1. **`.ai/skills/architecture.md`** — folder structure, class naming (every class has a type suffix: Action, Query, Service, Dto, Event, Listener, Job, Resource), FormRequest + DTO + Resource triad, layered architecture (Actions/Queries/Services), `final` rule, dependency direction, exception handling, model lifecycle, state machines, and a list of forbidden patterns
2. **`.ai/skills/testing.md`** — Pest framework, Postgres + RefreshDatabase, feature vs unit test boundaries, `Http::fake()` for HTTP services, interface + Fake class pattern for SDK services, factory patterns, what NOT to test

## Key rules

- Every concrete class is `final` (unless abstract)
- Class names carry their type suffix: `PlaceOrderAction`, `OrderPlacedEvent`, `CreateOrderDto`, `OrderResource`
- HTTP endpoints: FormRequest validates, Controller constructs DTO via `Dto::from($request->validated())`, Action runs the operation, JsonResource serializes the response
- Business logic NEVER lives in controllers, models, lifecycle hooks, or DTOs
- DTOs use `public readonly` properties, no validation attributes, `spatie/laravel-data` base class
- Actions fire Events AFTER `DB::transaction()` commits
- Listeners implement `ShouldQueue` by default
- All Services get the `Service` suffix uniformly
- Tests: Pest framework, real Postgres, `RefreshDatabase` trait, parallel execution
- The forbidden-patterns list in `architecture.md` is non-negotiable

## Generation conventions

When generating files:

- Include `<?php`, `declare(strict_types=1);`, full namespace, all `use` statements
- Show complete files, never partial snippets unless explicitly asked
- For a new write endpoint, generate: FormRequest + Dto + Action + Resource (if new) + Controller method + route + tests

## Human-readable standards

For the full standards with rationale, see `README.md`, `04-architecture-patterns.md`, `09-testing-strategy.md`, and the `architecture/` folder.
