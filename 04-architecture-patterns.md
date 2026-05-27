# Section 4: Architecture Patterns

## Goal

Define a single, consistent structure for organizing business logic, data flow, and side effects. Every developer knows where to put a new piece of code and where to find existing ones.

## Core Layers

```
Controller / Job / Console Command           ← entry points (thin)
        ↓                     ↓
   Actions (writes)       Queries (reads)
        ↓                     ↓
                Services
                    ↓
        Models / HTTP / Filesystem
```

For deep reference on each layer, see [`architecture/`](./architecture/).

## Folder Structure

```
app/
├── Actions/                       # Synchronous business operations
│   └── Orders/
│       ├── PlaceOrder.php
│       ├── CancelOrder.php
│       └── MarkOrderAsShipped.php
├── Queries/                       # Read operations with complexity
│   └── Orders/
│       ├── ListOrdersQuery.php
│       └── OrderStatsQuery.php
├── Services/                      # Capability wrappers (external APIs, libraries)
│   ├── SmsService.php
│   ├── StripeService.php
│   └── OrderNumberGenerator.php
├── Data/                          # spatie/laravel-data classes
│   └── Orders/
│       ├── CreateOrderData.php
│       ├── UpdateOrderData.php
│       └── OrderData.php
├── Jobs/                          # Async work (dispatched, not called)
├── Events/
├── Listeners/                     # Queued reactions to events
├── Concerns/                      # Model traits with boot{Name}() hooks
│   ├── HasUuid.php
│   └── HasOrderNumber.php
├── Models/
│   └── Order/
│       ├── Order.php
│       └── States/                # spatie/model-states (co-located with model)
│           ├── OrderState.php     # abstract base
│           ├── Pending.php
│           ├── Shipped.php
│           └── Transitions/
│               └── PendingToShipped.php
└── Http/
    ├── Controllers/               # Thin: validate → dispatch → respond
    └── Requests/                  # Authorization only
```

## Rules

### 1. Business Logic Placement

- **Writes** → Action class in `app/Actions/`
- **Reads with complexity** (3+ filters, multiple joins, reused) → Query class in `app/Queries/`
- **External APIs / libraries / capabilities** → Service class in `app/Services/`
- **Simple reads** (`Model::find($id)`) → controller direct
- Logic NEVER lives in controllers, models, observers, or lifecycle hooks.

### 2. Class Shape

- Actions: one public method (`handle()`), one use case per class
- Queries: one public method (`handle()`), read-only, no side effects
- Services: multiple methods OK if cohesive around one capability/external system
- All dependencies via constructor injection. No `app()` or `resolve()` inside methods.

### 3. `final` By Default

All concrete classes in:
- `app/Actions/`
- `app/Queries/`
- `app/Services/`
- `app/Data/`
- `app/Jobs/`
- `app/Listeners/`
- `app/Observers/` (if used)
- `app/Http/Middleware/`
- `app/Models/{Model}/States/`
- `app/Models/{Model}/States/Transitions/`

Rules:
- Abstract bases use `abstract`, not `final`
- Removing `final` requires a `// @reason ...` comment at the class declaration
- Rector rule `FinalizeClassesWithoutChildrenRector` enforces this in CI
- Services that need test doubles use **interfaces**, not subclass mocking

### 4. Validation & Data Flow

- **`spatie/laravel-data`** for input validation, output serialization, and inter-layer DTOs
- TypeScript generation mandatory: `php artisan typescript:transform` runs in CI
- **Form Requests for authorization only** (`authorize()` method)
- Validation rules live in Data classes via attributes
- Naming:
  - `Create{X}Data` — input for creation
  - `Update{X}Data` — input for updates (uses `Optional` for partial fields)
  - `{X}Data` — output / between layers

### 5. No Repository Pattern

- Eloquent IS the data access layer
- Complex reads → Query class (still uses Eloquent internally)
- External data sources → Service class

### 6. Model Lifecycle Hooks

- Use **traits with `boot{Name}()` pattern** in `app/Concerns/`
- One trait per concern (`HasUuid`, `HasOrderNumber`, `Auditable`)
- Models declare which behaviors via `use HasUuid;`
- No `boot()` / `booted()` directly in models
- No standalone Observer classes (exception requires documented justification)
- Hooks do **plumbing only**: UUIDs, slugs, audit logs, cascade cleanups, model invariants
- No business logic, side effects (SMS/email/webhooks), or external calls in hooks — those belong in Actions

### 7. Events & Side Effects

- **Actions fire Events** (`event(new SomethingHappened(...))`) for fan-out side effects
- **All Listeners implement `ShouldQueue` by default**
- One Listener per reaction (`SendOrderConfirmationSms`, not `OrderPlacedHandler` doing 5 things)
- Actions dispatch Jobs directly only for:
  - **Delayed** work (`->delay()`)
  - **Scheduled** work
  - **Chained** or **batched** jobs requiring strict ordering
- Actions are **synchronous** — they do NOT implement `ShouldQueue`
- Jobs in `app/Jobs/` are async, dispatched (never called directly except in tests)
- Jobs MAY fire Events from their `handle()` method (bulk imports, webhook processors, scheduled work)

### 8. State Machines

- `spatie/laravel-model-states` for any model with 3+ states or transition rules
- State classes co-located with model: `app/Models/{Model}/States/`
- Transition classes (one per non-trivial transition) in `app/Models/{Model}/States/Transitions/`
- Transition does atomic state change only (set fields, save, in transaction)
- Action class orchestrates the use case (calls transition + fires event)
- Simple status fields (active/inactive booleans) don't need state machines

### 9. Dependency Direction

Allowed:
- Controllers → Actions, Queries, Services
- Actions → Queries, Services, other Actions (sparingly), Events, Jobs (delayed only)
- Queries → Services (rare, e.g. search indexes)
- Listeners → Services
- Services → Models, HTTP clients, filesystem

Forbidden:
- Queries CANNOT call Actions
- Services CANNOT call Actions or Queries
- Listeners SHOULD NOT fire events that trigger their own listener chain
- Models SHOULD NOT contain business logic, only data definitions, casts, relationships

## See Also

- [`architecture/README.md`](./architecture/README.md) — index of architectural building blocks
- [`architecture/01-action.md`](./architecture/01-action.md) — Actions
- [`architecture/02-query.md`](./architecture/02-query.md) — Queries
- [`architecture/03-service.md`](./architecture/03-service.md) — Services
- [`architecture/04-dto.md`](./architecture/04-dto.md) — DTOs / Data classes
- [`architecture/05-event.md`](./architecture/05-event.md) — Events
- [`architecture/06-listener.md`](./architecture/06-listener.md) — Listeners
- [`architecture/07-job.md`](./architecture/07-job.md) — Jobs
- [`architecture/08-controller.md`](./architecture/08-controller.md) — Controllers
- [`architecture/09-form-request.md`](./architecture/09-form-request.md) — Form Requests
- [`architecture/10-model-trait.md`](./architecture/10-model-trait.md) — Model Lifecycle Traits
- [`architecture/11-state-and-transition.md`](./architecture/11-state-and-transition.md) — States & Transitions
- [`architecture/12-model.md`](./architecture/12-model.md) — Models
