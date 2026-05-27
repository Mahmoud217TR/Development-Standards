# Section 9: Testing Strategy

> **Status:** 🔄 In progress — 8 open questions awaiting decisions.

## Goal

Decide the team's testing framework, philosophy, coverage requirements, and tooling. Given the architecture locked in Section 4 (Actions, Queries, Services, Data, Events, Listeners, Jobs, State machines), define what gets tested at what layer and how.

## Architecture mapping

The testable surface from Section 4:

| Layer | Best tested via |
|---|---|
| **Actions** | Unit tests — pure orchestration with mockable dependencies |
| **Queries** | Unit tests with database |
| **Services** | Unit tests with fakes/mocks for external calls |
| **Data classes** | Unit tests for validation, casting, transformation |
| **Controllers** | Feature tests (full HTTP round-trip) |
| **Listeners** | Unit tests for `handle()`, dispatch assertions in feature tests |
| **Jobs** | Unit tests for `handle()`, dispatch assertions in feature tests |
| **State transitions** | Unit tests for transition classes; feature tests for state-change endpoints |
| **Model traits** | Tested implicitly via feature tests; explicit unit tests for non-trivial logic |

## Open questions

1. Test framework? (Pest / PHPUnit / Mixed)
2. Testing philosophy? (Architecture-driven split / Feature-only / Pyramid / Balanced)
3. Coverage requirement? (None / 70% soft / 80% differential / 80% global)
4. When are tests mandatory? (Every PR / Features + bug fixes / Critical paths / Encouraged)
5. Database strategy? (RefreshDatabase / DatabaseMigrations / Hybrid / Real Postgres in Docker)
6. Browser / E2E tests? (None / Later / Playwright from day one / Dusk)
7. Factories for Data classes? (Data::fake() / Eloquent factories converted / Hand-craft)
8. External service strategy? (Mock via interfaces / Http::fake() + Mail::fake() / Mix)

## Next step

Answer the 8 questions, then this section gets fully written up with locked rules.
