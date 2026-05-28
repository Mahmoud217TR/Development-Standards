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

The HTTP layer uses **three distinct tools** for the request/response cycle:

- **FormRequest** — authorization + validation
- **DTO** — typed transport of validated data between layers
- **JsonResource** — JSON output serialization

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
├── Data/                          # Plain DTOs (spatie/laravel-data Data classes — no validation)
│   └── Orders/
│       ├── CreateOrderDto.php
│       ├── UpdateOrderDto.php
│       └── OrderFiltersDto.php
├── Jobs/                          # Async work (dispatched, not called)
├── Events/
├── Listeners/                     # Queued reactions to events
├── Exceptions/                    # Domain exception classes
├── Concerns/                      # Reusable model traits with boot{Name}() hooks
│   ├── HasUuid.php
│   └── Auditable.php
├── Models/
│   └── Order/
│       ├── Order.php              # boot()/booted() hooks live HERE
│       └── States/
│           ├── OrderState.php     # abstract base
│           ├── Pending.php
│           ├── Shipped.php
│           └── Transitions/
│               └── PendingToShipped.php
└── Http/
    ├── Controllers/               # Thin: validate → dispatch → respond
    ├── Requests/                  # FormRequests: authorize + validate
    │   └── Orders/
    │       ├── StoreOrderRequest.php
    │       └── UpdateOrderRequest.php
    └── Resources/                 # JsonResources: output serialization
        ├── OrderResource.php
        └── OrderItemResource.php
```

## Rules

### 1. Business Logic Placement

- **Writes** → Action class in `app/Actions/`
- **Reads with complexity** (3+ filters, multiple joins, reused) → Query class in `app/Queries/`
- **External APIs / libraries / capabilities** → Service class in `app/Services/`
- **Simple reads** (`Model::find($id)`) → controller direct
- Logic NEVER lives in controllers or hooks. Models hold data shape only.

### 2. Class Shape

- Actions, Queries: one public method named `handle()`
- Services: multiple methods OK if cohesive around one capability/external system
- All dependencies via **constructor injection only**

**`app()` / `resolve()` rule:**
- **Forbidden** inside method bodies of classes that have a constructor (Actions, Queries, Services, Jobs, Listeners, Controllers, Middleware, Form Requests, DTOs)
- **Permitted** in contexts without DI: model `boot{Name}()` static closures, route closures, factory `state()` callbacks

### 3. `final` By Default

All concrete classes in:
- `app/Actions/`, `app/Queries/`, `app/Services/`
- `app/Data/` (DTOs)
- `app/Jobs/`, `app/Listeners/`
- `app/Http/Middleware/`, `app/Http/Requests/`, `app/Http/Resources/`, `app/Http/Controllers/`
- `app/Models/{Model}/States/` and `app/Models/{Model}/States/Transitions/`
- `app/Exceptions/`

Rules:
- Abstract bases use `abstract`, not `final`
- Removing `final` requires a `// @reason ...` comment at the class declaration
- Rector rule `FinalizeClassesWithoutChildrenRector` enforces this in CI
- Services that need test doubles use **interfaces**, not subclass mocking

### 4. HTTP I/O Tools — Three Layers

**FormRequest** — `app/Http/Requests/`
- Handles **authorization** (`authorize()`) AND **validation** (`rules()`, `messages()`)
- One FormRequest per write endpoint
- Naming: `{Verb}{Object}Request` — `StoreOrderRequest`, `UpdateOrderRequest`, `UpdateUserEmailRequest`

**DTO** — `app/Data/`
- Plain typed transport class using `spatie/laravel-data` for `::from()` ergonomics
- **NO validation attributes** (validation belongs in FormRequest)
- **NO TypeScript attributes** (frontend types maintained separately)
- All properties `public readonly`
- All DTOs `final`
- Naming:
  - `Create{X}Dto` — input for creation
  - `Update{X}Dto` — input for update; OR `Update{X}{Field}Dto` for per-field-group updates (e.g., `UpdateUserNameDto`, `UpdateUserEmailDto`)
  - `{X}FiltersDto` — query parameters for Queries
  - `{X}Dto` — generic transport between layers (when needed)

**JsonResource** — `app/Http/Resources/`
- Handles **output serialization**
- Naming: `{X}Resource` — `OrderResource`, `UserResource`
- Uses `when()`, `whenLoaded()`, `whenCounted()` for conditional output
- Collection responses use `{X}Resource::collection($items)`

### 5. The flow on every endpoint

**Write endpoint:**
```php
public function store(StoreOrderRequest $request, PlaceOrder $action)
{
    $dto = CreateOrderDto::from($request->validated());
    $order = $action->handle($dto);
    return new OrderResource($order);
}
```

1. FormRequest authorizes (403 if denied)
2. FormRequest validates (422 if invalid)
3. Controller constructs DTO from validated data
4. Controller invokes Action with DTO
5. Action returns Model
6. Controller wraps Model in Resource
7. Laravel serializes Resource to JSON

**Read endpoint:**
```php
public function index(Request $request, ListOrdersQuery $query)
{
    $filters = OrderFiltersDto::from($request->query());
    return OrderResource::collection($query->handle($filters));
}
```

### 6. Per-field-group update pattern

When an entity has distinct update flows (different validation, different side effects), use separate FormRequests + DTOs + Actions per flow. Example for User:

```
app/Http/Requests/Users/
  ├── UpdateUserNameRequest.php
  ├── UpdateUserEmailRequest.php
  └── UpdateUserAddressRequest.php

app/Data/Users/
  ├── UpdateUserNameDto.php
  ├── UpdateUserEmailDto.php
  └── UpdateUserAddressDto.php

app/Actions/Users/
  ├── UpdateUserName.php
  ├── UpdateUserEmail.php      ← may trigger re-verification
  └── UpdateUserAddress.php

app/Http/Resources/
  └── UserResource.php         ← single output Resource

Routes:
  PATCH /profile/name     → ProfileController@updateName
  PATCH /profile/email    → ProfileController@updateEmail
  PATCH /profile/address  → ProfileController@updateAddress
```

Use a **single combined `UpdateUserDto`** with optional fields **only** when one form legitimately edits all fields together (e.g., admin "edit user" page).

### 7. No Repository Pattern

- Eloquent IS the data access layer
- Complex reads → Query class (still uses Eloquent internally)
- External data sources → Service class

### 8. Model Lifecycle Hooks

- Primary mechanism: **`boot()` / `booted()` methods directly in the model class**
- Cross-cutting concerns (UUIDs across many models, audit logging) → trait with `boot{Name}()` in `app/Concerns/`
- Observer classes are **not used**
- Hooks do **plumbing only**: UUIDs, slugs, audit logs, cascade cleanups, model invariants
- No business logic, side effects (SMS/email/webhooks), or external calls in hooks — those belong in Actions

### 9. Events & Side Effects — Sync vs Async

**The decision rule:** "Does the HTTP response need the result of this work?"

- **Yes** → work runs **synchronously inside the Action**. Inside `DB::transaction()` if it modifies the database. The Action returns the enriched result.
- **No** → work runs **asynchronously after the Action**. Either via Events (fan-out reactions) or directly dispatched Jobs (delayed/scheduled/known reactions).

**Examples:**
- Calculating loyalty points awarded by an order, where the response needs to show "you earned 150 points" → synchronous, inside the Action.
- Sending a confirmation SMS — frontend doesn't need to know the SMS sent → async, via queued Listener.
- Generating an invoice PDF for download in the response → synchronous. Async only if "PDF will be emailed to you."

**Rules:**
- Events fired **AFTER** `DB::transaction()` commits
- All Listeners implement `ShouldQueue` by default
- One Listener per reaction
- Actions dispatch Jobs directly only for: delayed work (`->delay()`), scheduled work, chained/batched jobs
- Actions are **synchronous** — never implement `ShouldQueue`
- Jobs in `app/Jobs/` are async, dispatched (never called directly except in tests)
- Jobs MAY fire Events from inside `handle()`

### 10. State Machines

- `spatie/laravel-model-states` for any model with 3+ states or transition rules
- State classes co-located with model: `app/Models/{Model}/States/`
- Transition classes (one per non-trivial transition) in `app/Models/{Model}/States/Transitions/`
- Transition does atomic state change only
- Action class orchestrates the use case (calls transition + fires event)
- Simple status fields (active/inactive booleans) don't need state machines

### 11. Exception Handling

Detailed in [`architecture/13-exception.md`](./architecture/13-exception.md). Summary:

- **Throw at the source.** Services translate vendor exceptions to domain exceptions. Actions throw domain exceptions for business-rule violations.
- **Don't catch in Queries, Controllers, or business code.** Let exceptions propagate.
- **Catch only when**: (a) a Service is translating to a domain exception, or (b) an Action has an explicit business fallback.
- **Centralize HTTP translation** in `bootstrap/app.php` (Laravel 11+) or `App\Exceptions\Handler` (Laravel 10).
- **Domain exceptions** live in `app/Exceptions/`, one class per failure type.
- **Jobs/Listeners** don't catch in `handle()`; rely on `$tries`, `$backoff`, `failed()`.
- **Never catch `\Exception` or `\Throwable`** in business code.

### 12. Dependency Direction

Allowed:
- Controllers → FormRequests (auto-resolved), DTOs (constructed inside), Actions, Queries, Resources
- Actions → Queries, Services, other Actions (sparingly), Events, Jobs (delayed only)
- Queries → Services (rarely)
- Listeners → Services
- Services → Models, HTTP clients, filesystem

Forbidden:
- Queries CANNOT call Actions
- Services CANNOT call Actions or Queries
- DTOs CANNOT have methods other than the constructor and `::from()` (no business logic on DTOs)
- Resources CANNOT call Actions or query the database (output formatting only)
- Listeners SHOULD NOT fire events that trigger their own listener chain
- Models hold data definitions, casts, relationships, lifecycle hooks — no business logic

## See Also

- [`architecture/README.md`](./architecture/README.md) — index of architectural building blocks
- [`architecture/01-action.md`](./architecture/01-action.md) — Actions
- [`architecture/02-query.md`](./architecture/02-query.md) — Queries
- [`architecture/03-service.md`](./architecture/03-service.md) — Services
- [`architecture/04-dto.md`](./architecture/04-dto.md) — DTOs
- [`architecture/05-event.md`](./architecture/05-event.md) — Events
- [`architecture/06-listener.md`](./architecture/06-listener.md) — Listeners
- [`architecture/07-job.md`](./architecture/07-job.md) — Jobs
- [`architecture/08-controller.md`](./architecture/08-controller.md) — Controllers
- [`architecture/09-form-request.md`](./architecture/09-form-request.md) — Form Requests
- [`architecture/10-model-trait.md`](./architecture/10-model-trait.md) — Model Lifecycle
- [`architecture/11-state-and-transition.md`](./architecture/11-state-and-transition.md) — States & Transitions
- [`architecture/12-model.md`](./architecture/12-model.md) — Models
- [`architecture/13-exception.md`](./architecture/13-exception.md) — Exceptions
- [`architecture/14-resource.md`](./architecture/14-resource.md) — JsonResources
