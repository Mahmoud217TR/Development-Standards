# Architecture Skill

You are working in a Laravel codebase that follows a **strict, opinionated architecture**. These rules are mandatory. Every file you create or modify must conform.

When unsure between options, choose the one that produces the most explicit, type-safe, testable code.

---

## The triad — every HTTP endpoint uses three tools

Every write endpoint flows through three distinct classes, each with one job:

| Tool | Job | Lives in | Suffix |
|---|---|---|---|
| **FormRequest** | Authorize + Validate input | `app/Http/Requests/` | `Request` |
| **DTO** | Carry validated data between layers | `app/Data/` | `Dto` |
| **JsonResource** | Format Model → JSON output | `app/Http/Resources/` | `Resource` |

Validation NEVER lives in DTOs or Controllers. Output formatting NEVER lives in DTOs or Controllers. Authorization NEVER lives in Actions or Controllers.

## The layers

```
Controller / Job / Console Command           ← entry points (thin)
        ↓                     ↓
   Actions (writes)       Queries (reads)
        ↓                     ↓
                Services
                    ↓
        Models / HTTP / Filesystem
```

**Dependency direction is strict — pointers only go down or sideways within a tier:**

- Controllers → Actions, Queries, FormRequests, DTOs, Resources
- Actions → Services, Models, Queries, sub-Actions (sparingly), Events, Jobs (delayed/scheduled only)
- Queries → Models, Services (rare; e.g., search index)
- Listeners → Services, Jobs
- Services → Models, HTTP clients, filesystem

**FORBIDDEN:**
- Queries calling Actions
- Services calling Actions or Queries
- Listeners firing events that trigger their own listener chain
- Models containing business logic
- Resources accessing the database
- Controllers building queries or applying business rules

---

## Naming convention — class suffixes are MANDATORY

Every class of a known type carries the type as its suffix.

| Type | Suffix | Example |
|---|---|---|
| Action | `Action` | `PlaceOrderAction` |
| Query | `Query` | `ListUserOrdersQuery` |
| Service | `Service` | `SmsService`, `OrderNumberGeneratorService` |
| DTO | `Dto` | `CreateOrderDto` |
| Event | `Event` | `OrderPlacedEvent` |
| Listener | `Listener` | `SendOrderConfirmationSmsListener` |
| Job | `Job` | `ImportProductsFromCsvJob` |
| JsonResource | `Resource` | `OrderResource` |
| Controller | `Controller` | `OrderController` |
| FormRequest | `Request` | `StoreOrderRequest` |
| Domain exception | `Exception` | `InsufficientInventoryException` |

**No exceptions.** Even capability-named Services get the `Service` suffix uniformly: `OrderNumberGeneratorService`, `PdfRendererService`, `MoneyFormatterService`.

Models, Concerns (traits), States, and Transitions do NOT take suffixes — they're disambiguated by location.

---

## Folder structure — flat by default, subfolders past 8 files

For each type, keep files flat in their root folder until the count reaches **8**. At that point, ALL files of that type move into domain subfolders in a single refactor PR. Mixed flat + nested for the same type is NOT allowed.

```
app/
├── Actions/                          # flat if < 8 Actions
│   ├── PlaceOrderAction.php
│   └── RegisterUserAction.php
│   OR (once 8+ files):
│   ├── Orders/
│   │   ├── PlaceOrderAction.php
│   │   └── CancelOrderAction.php
│   └── Users/
│       └── RegisterUserAction.php
├── Queries/                          # same rule
├── Services/                          # same rule (group by capability area, not domain)
├── Data/                              # same rule
├── Events/                            # same rule
├── Listeners/                         # ALWAYS flat (exception to the 8+ rule)
├── Jobs/                              # same rule
├── Exceptions/                        # same rule
├── Concerns/                          # traits with boot{Name}() hooks
├── Models/
│   └── {Model}/
│       ├── {Model}.php                # boot()/booted() hooks live HERE
│       └── States/                    # spatie/model-states
│           ├── {Model}State.php
│           ├── Pending.php
│           └── Transitions/
│               └── PendingToShipped.php
└── Http/
    ├── Controllers/                   # flat + Auth/, Admin/ subfolders OK
    ├── Requests/                      # same rule
    └── Resources/                     # same rule
```

When promoting to subfolders, name them PascalCase, plural where natural, domain-based (`Orders/`, `Users/`, `Payments/`) or functional (`Admin/`, `Auth/`). Never verb-based (`Importing/`), never type-repeated (`OrderActions/`).

---

## Rule 1 — `final` by default

Every concrete class in these folders is `final`:

- `app/Actions/`, `app/Queries/`, `app/Services/`, `app/Data/`
- `app/Jobs/`, `app/Listeners/`
- `app/Http/Controllers/`, `app/Http/Middleware/`, `app/Http/Requests/`, `app/Http/Resources/`
- `app/Models/{Model}/States/` and its `Transitions/`
- `app/Exceptions/`

Abstract bases use `abstract`. To extend instead of compose, you must write `// @reason ...` at the class declaration.

To enable mocking in tests, define an **interface** and have the Service implement it. Never make the class non-final to enable subclass mocking.

```php
interface SmsServiceContract { /* ... */ }
final class SmsService implements SmsServiceContract { /* ... */ }
final class FakeSmsService implements SmsServiceContract { /* tests/Fakes/ */ }
```

## Rule 2 — Constructor injection only; never `app()` or `resolve()` inside method bodies

**Forbidden in classes with constructors** (Actions, Queries, Services, Jobs, Listeners, Controllers, Middleware, FormRequests, DTOs):

```php
public function handle(CreateOrderDto $dto): Order
{
    $sms = app(SmsService::class);  // ❌ NO
}
```

**Required:**

```php
public function __construct(
    private SmsService $sms,
) {}

public function handle(CreateOrderDto $dto): Order
{
    $this->sms->send(...);  // ✅
}
```

**Permitted** in contexts without DI: model `boot{Name}()` closures, route closures, factory `state()` callbacks.

---

## Action — write use case

**File:** `app/Actions/{Domain}/{Verb}{Object}Action.php`
**Method:** `handle()` (single public method)
**Returns:** Model, or a Dto if returning composite data

### Skeleton

```php
<?php

declare(strict_types=1);

namespace App\Actions\Orders;

use App\Data\Orders\CreateOrderDto;
use App\Events\OrderPlacedEvent;
use App\Exceptions\InsufficientInventoryException;
use App\Models\Order\Order;
use App\Services\InventoryService;
use App\Services\OrderNumberGeneratorService;
use Illuminate\Support\Facades\DB;

final class PlaceOrderAction
{
    public function __construct(
        private InventoryService $inventory,
        private OrderNumberGeneratorService $numbers,
    ) {}

    public function handle(CreateOrderDto $dto): Order
    {
        // 1. preconditions → throw domain exception
        if (!$this->inventory->isAvailable($dto->items)) {
            throw new InsufficientInventoryException(
                $this->inventory->getUnavailable($dto->items)
            );
        }

        // 2. DB::transaction opens
        $order = DB::transaction(function () use ($dto) {
            $order = Order::create([
                'number' => $this->numbers->next(),
                'customer_name' => $dto->customer_name,
                'phone' => $dto->phone,
            ]);

            $order->items()->createMany($dto->items->toArray());

            // 3. sync work the response needs goes inside the transaction
            return $order;
        });

        // 4. fire Event AFTER transaction commits
        event(new OrderPlacedEvent($order));

        // 5. return result
        return $order;
    }
}
```

### Action rules

1. Exactly one public method: `handle()`
2. Synchronous — never `implements ShouldQueue`
3. All persistence inside `DB::transaction()` when 2+ writes are involved
4. Events fire AFTER the transaction commits (otherwise queued listeners read uncommitted data)
5. Returns Model (preferred) or Dto (for composite results)
6. Never accepts `Request`; accepts a Dto
7. `final`
8. Throws domain exceptions for business-rule violations
9. Catching is forbidden except for explicit business fallback (with `// @reason ...` comment)

---

## Query — read use case

**File:** `app/Queries/{Domain}/{Verb}{Object}Query.php`
**Method:** `handle()`
**Returns:** Collection / LengthAwarePaginator / Dto / Model

### Skeleton

```php
<?php

declare(strict_types=1);

namespace App\Queries\Orders;

use App\Data\Orders\OrderFiltersDto;
use App\Models\User;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;

final class ListUserOrdersQuery
{
    public function handle(User $user, OrderFiltersDto $filters): LengthAwarePaginator
    {
        return $user->orders()
            ->when($filters->status, fn ($q, $status) => $q->where('status', $status))
            ->when($filters->search, fn ($q, $search) => $q->where(function ($q) use ($search) {
                $q->where('number', 'like', "%{$search}%")
                  ->orWhere('customer_name', 'like', "%{$search}%");
            }))
            ->when($filters->date_from, fn ($q, $from) => $q->where('created_at', '>=', $from))
            ->when($filters->date_to, fn ($q, $to) => $q->where('created_at', '<=', $to))
            ->with(['items'])
            ->latest()
            ->paginate($filters->per_page ?? 20);
    }
}
```

### Query rules

1. **Read-only.** Never writes, never fires events, never has side effects
2. Exactly one public method: `handle()`
3. Returns typed result; never raw arrays
4. Accepts filter Dto / domain model / primitive — never `Request`
5. Never calls Actions
6. Never uses `auth()` inside — pass user as parameter so it's usable from jobs/console
7. `final`
8. Use a Query class only when read complexity justifies it: 3+ filters, multiple joins, aggregations, or reused across endpoints. Trivial `Model::find($id)` stays inline in the controller.

---

## Service — capability wrapper

**File:** `app/Services/{Subject}{Capability}Service.php`
**Methods:** Multiple public methods OK if cohesive around one capability/external system
**Returns:** Varies by method

### Skeleton — external API with exception translation

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\Data\Payments\RefundDto;
use App\Exceptions\PaymentDeclinedException;
use App\Exceptions\PaymentGatewayException;

final class StripeService implements PaymentGatewayContract
{
    public function __construct(
        private \Stripe\StripeClient $client,
    ) {}

    public function refund(string $chargeId, int $amount): RefundDto
    {
        try {
            $response = $this->client->refunds->create([
                'charge' => $chargeId,
                'amount' => $amount,
            ]);
        } catch (\Stripe\Exception\CardException $e) {
            throw new PaymentDeclinedException($e->getMessage(), previous: $e);
        } catch (\Stripe\Exception\ApiErrorException $e) {
            throw new PaymentGatewayException($e->getMessage(), previous: $e);
        }

        return RefundDto::from($response);
    }
}
```

### Skeleton — internal capability

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\Models\OrderSequence;

final class OrderNumberGeneratorService
{
    public function next(): string
    {
        $year = now()->year;
        $sequence = OrderSequence::increment($year);
        return sprintf('ORD-%d-%05d', $year, $sequence);
    }
}
```

### Service rules

1. **No business logic.** Services don't decide *whether* to send SMS; they send when asked
2. Services do not fire events. Orchestration lives in Actions
3. Services do not call Actions or Queries
4. **Catching is allowed when translating vendor exceptions to domain exceptions.** Never catch and swallow
5. `final`; if the Service needs a test double, define an interface
6. Pragmatic naming — `Service` suffix mandatory even when awkward (`PdfRendererService`)

---

## DTO — typed transport

**File:** `app/Data/{Domain}/{Name}Dto.php`
**Built on:** `spatie/laravel-data` (for `::from()` only — NO validation attributes, NO TypeScript attributes)
**Properties:** `public readonly` always

### Naming

- `Create{X}Dto` — input for creation (POST)
- `Update{X}Dto` — input for update; OR `Update{X}{Field}Dto` for per-field-group updates
- `{X}FiltersDto` — query parameters for Queries
- `{X}Dto` — generic transport between layers

### Skeleton

```php
<?php

declare(strict_types=1);

namespace App\Data\Orders;

use Spatie\LaravelData\Data;

final class CreateOrderDto extends Data
{
    public function __construct(
        public readonly string $customer_name,
        public readonly string $phone,
        /** @var array<OrderItemDto> */
        public readonly array $items,
        public readonly ?string $notes = null,
    ) {}
}
```

### Filter DTO

```php
<?php

declare(strict_types=1);

namespace App\Data\Orders;

use Carbon\Carbon;
use Spatie\LaravelData\Data;

final class OrderFiltersDto extends Data
{
    public function __construct(
        public readonly ?string $status = null,
        public readonly ?string $search = null,
        public readonly ?Carbon $date_from = null,
        public readonly ?Carbon $date_to = null,
        public readonly ?string $sort = null,
        public readonly ?string $direction = 'desc',
        public readonly int $per_page = 20,
    ) {}
}
```

### DTO rules

1. **No validation.** No `#[Required]`, `#[Email]`, etc. — validation lives in FormRequests
2. **No methods.** Only the constructor and inherited `::from()`. No business logic, no calculations
3. **No TypeScript attributes.** Frontend types maintained separately
4. All properties `public readonly`
5. `final`
6. Suffix is `Dto`, never `Data`

---

## Event — past-tense fact

**File:** `app/Events/{X}Event.php`
**Naming:** Past tense — `OrderPlacedEvent`, `UserRegisteredEvent`. Never imperative
**Properties:** Carry the domain entity, not raw fields

### Skeleton

```php
<?php

declare(strict_types=1);

namespace App\Events;

use App\Models\Order\Order;

final class OrderPlacedEvent
{
    public function __construct(
        public readonly Order $order,
    ) {}
}
```

### Event rules

1. Named in past tense + `Event` suffix
2. Carry the domain entity (not raw fields)
3. Public readonly properties only
4. No methods beyond the constructor
5. `final`
6. Fired only by Actions and Jobs, AFTER any database transaction commits
7. Listeners decide what to do — Events don't

---

## Listener — queued reaction

**File:** `app/Listeners/{VerbObject}Listener.php` (flat folder always)
**Naming:** Verb phrase describing what it does — `SendOrderConfirmationSmsListener`, NOT `OrderPlacedListener`
**Method:** `handle()` taking the Event

### Skeleton

```php
<?php

declare(strict_types=1);

namespace App\Listeners;

use App\Events\OrderPlacedEvent;
use App\Services\SmsService;
use Illuminate\Contracts\Queue\ShouldQueue;

final class SendOrderConfirmationSmsListener implements ShouldQueue
{
    public int $tries = 3;
    public int $backoff = 60;
    public string $queue = 'notifications';

    public function __construct(
        private SmsService $sms,
    ) {}

    public function handle(OrderPlacedEvent $event): void
    {
        $this->sms->send(
            $event->order->phone,
            "Order {$event->order->number} received. Thank you!",
        );
    }
}
```

### Listener rules

1. **One Listener per reaction.** Don't combine SMS + email + webhook in one Listener
2. `implements ShouldQueue` by default
3. Exactly one public method: `handle({EventClass} $event): void`
4. Named after what it does (verb phrase), not what it reacts to
5. Listeners may use Services. They MUST NOT call Actions
6. Listeners SHOULD NOT fire further events (avoid chains)
7. Never catch inside `handle()` — let the queue's retry mechanism (`$tries`, `$backoff`, `failed()`) own failure
8. `final`
9. Listeners stay in `app/Listeners/` flat — no subfolders even past 8 files

---

## Job — async work

**File:** `app/Jobs/{VerbObject}Job.php`
**Naming:** Verb phrase + `Job` suffix
**Method:** `handle()` with injected dependencies

### Skeleton

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use App\Events\ProductsImportedEvent;
use App\Models\User;
use App\Services\CsvParserService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

final class ImportProductsFromCsvJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 1;
    public int $timeout = 600;
    public string $queue = 'imports';

    public function __construct(
        public string $filePath,
        public User $merchant,
    ) {}

    public function handle(CsvParserService $parser): void
    {
        $imported = collect();
        foreach ($parser->rows($this->filePath) as $row) {
            $product = $this->merchant->products()->create($row);
            $imported->push($product);
        }

        event(new ProductsImportedEvent($this->merchant, $imported));
    }

    public function failed(\Throwable $e): void
    {
        report($e);
    }
}
```

### Job rules

1. `implements ShouldQueue` (always)
2. Dispatched via `Job::dispatch(...)`, never called directly except in tests
3. Constructor args must be **serializable** (primitives, Models via `SerializesModels`, Dtos, file paths). NOT raw file handles or HTTP clients
4. `handle()` injects Services via type hints
5. Jobs MAY fire Events from `handle()` (e.g., bulk imports announcing completion)
6. Jobs SHOULD NOT call Actions directly — if they would, reconsider whether the work belongs as an Action+Job split
7. `final`
8. Configure `$tries`, `$backoff`, `$timeout`, `$queue` per Job class
9. Never catch inside `handle()`; use `failed()` for final-failure logic

---

## Controller — thin HTTP adapter

**File:** `app/Http/Controllers/{Resource}Controller.php`
**Methods:** Each 3–6 lines after parameter declarations
**Returns:** JsonResource, JsonResponse, redirect, or `response()->noContent()`

### Skeleton — Resource controller

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Actions\Orders\PlaceOrderAction;
use App\Actions\Orders\UpdateOrderAction;
use App\Actions\Orders\CancelOrderAction;
use App\Data\Orders\CreateOrderDto;
use App\Data\Orders\UpdateOrderDto;
use App\Data\Orders\OrderFiltersDto;
use App\Http\Requests\Orders\StoreOrderRequest;
use App\Http\Requests\Orders\UpdateOrderRequest;
use App\Http\Requests\Orders\DestroyOrderRequest;
use App\Http\Resources\OrderResource;
use App\Models\Order\Order;
use App\Queries\Orders\ListUserOrdersQuery;
use Illuminate\Http\Request;

final class OrderController
{
    public function index(Request $request, ListUserOrdersQuery $query)
    {
        $filters = OrderFiltersDto::from($request->query());
        return OrderResource::collection(
            $query->handle($request->user(), $filters)
        );
    }

    public function show(Order $order)
    {
        return new OrderResource($order->load('items'));
    }

    public function store(StoreOrderRequest $request, PlaceOrderAction $action)
    {
        $dto = CreateOrderDto::from($request->validated());
        $order = $action->handle($dto);
        return new OrderResource($order->load('items'));
    }

    public function update(UpdateOrderRequest $request, Order $order, UpdateOrderAction $action)
    {
        $dto = UpdateOrderDto::from($request->validated());
        $order = $action->handle($order, $dto);
        return new OrderResource($order->load('items'));
    }

    public function destroy(DestroyOrderRequest $request, Order $order, CancelOrderAction $action)
    {
        $action->handle($order);
        return response()->noContent();
    }
}
```

### Skeleton — Invokable controller

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Orders;

use App\Actions\Orders\MarkOrderAsShippedAction;
use App\Data\Orders\ShipOrderDto;
use App\Http\Requests\Orders\ShipOrderRequest;
use App\Http\Resources\OrderResource;
use App\Models\Order\Order;

final class MarkOrderAsShippedController
{
    public function __invoke(
        ShipOrderRequest $request,
        Order $order,
        MarkOrderAsShippedAction $action,
    ) {
        $dto = ShipOrderDto::from($request->validated());
        $order = $action->handle($order, $dto);
        return new OrderResource($order);
    }
}
```

### Controller rules

1. **No business logic, no query building, no validation rules**
2. Each method 3–6 lines after parameter declarations
3. DTOs constructed **inside** the method body via `Dto::from($request->validated())` — NOT injected as parameters
4. Actions/Queries/FormRequests received via method-level DI
5. Resource controllers preferred for CRUD; invokable for non-CRUD endpoints
6. `final`

---

## FormRequest — authorize + validate

**File:** `app/Http/Requests/{Domain}/{Verb}{Object}Request.php`
**Methods:** `authorize()` and `rules()` at minimum

### Skeleton

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests\Orders;

use App\Models\Order\Order;
use Illuminate\Foundation\Http\FormRequest;

final class StoreOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()?->can('create', Order::class) ?? false;
    }

    public function rules(): array
    {
        return [
            'customer_name' => ['required', 'string', 'max:100'],
            'phone' => ['required', 'string', 'regex:/^09\d{8}$/'],
            'items' => ['required', 'array', 'min:1'],
            'items.*.product_id' => ['required', 'integer', 'exists:products,id'],
            'items.*.quantity' => ['required', 'integer', 'min:1'],
            'notes' => ['nullable', 'string', 'max:500'],
        ];
    }

    public function messages(): array
    {
        return [
            'phone.regex' => 'Phone must start with 09 and be 10 digits.',
            'items.required' => 'Order must contain at least one item.',
        ];
    }

    protected function prepareForValidation(): void
    {
        $this->merge([
            'phone' => str_replace([' ', '-'], '', (string) $this->phone),
            'customer_name' => trim((string) $this->customer_name),
        ]);
    }
}
```

### FormRequest rules

1. **Every write endpoint has a FormRequest**
2. `authorize()` returns bool (false → 403). Must NOT mutate state
3. `rules()` returns validation rules. Use array syntax: `['required', 'string', 'max:100']`
4. `messages()` for custom user-facing messages
5. `prepareForValidation()` for input normalization (trim, format phone, etc.)
6. No business logic
7. `final`

---

## JsonResource — output serialization

**File:** `app/Http/Resources/{X}Resource.php`
**Returns:** array from `toArray(Request $request)`

### Skeleton

```php
<?php

declare(strict_types=1);

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

final class OrderResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'number' => $this->number,
            'customer_name' => $this->customer_name,
            'phone' => $this->phone,
            'status' => $this->status->value,
            'total_amount' => $this->total_amount,
            'created_at' => $this->created_at->toIso8601String(),
            'items' => OrderItemResource::collection($this->whenLoaded('items')),
            'items_count' => $this->whenCounted('items'),
        ];
    }
}
```

### Conditional fields based on viewer

```php
public function toArray(Request $request): array
{
    $viewer = $request->user();
    $isOwn = $viewer?->id === $this->id;
    $isAdmin = $viewer?->isAdmin() ?? false;

    return [
        'id' => $this->id,
        'name' => $this->name,
        'email' => $this->when($isOwn || $isAdmin, $this->email),
        'phone' => $this->when($isOwn, $this->phone),
        $this->mergeWhen($isAdmin, [
            'last_login_at' => $this->last_login_at?->toIso8601String(),
        ]),
    ];
}
```

### Resource rules

1. **Output formatting only.** No business logic, no DB queries, no Action invocation
2. Use `when()`, `whenLoaded()`, `whenCounted()` for conditional fields — NOT `if/else`
3. Resources may FORMAT values (dates, money, enum→string), never COMPUTE them (totals with tax → compute in Action)
4. `final`
5. Lists return `XResource::collection($items)` — never instantiate Resource in a loop

---

## Domain Exception

**File:** `app/Exceptions/{Subject}{Failure}Exception.php`
**Extends:** `\DomainException` for business rules, `\RuntimeException` for infrastructure

### Skeleton

```php
<?php

declare(strict_types=1);

namespace App\Exceptions;

final class InsufficientInventoryException extends \DomainException
{
    public function __construct(
        public readonly array $unavailableItems,
    ) {
        parent::__construct('Some items are not available in the requested quantities.');
    }
}
```

### Exception handling rules

| Layer | Catches? |
|---|---|
| Database / external library | N/A — they throw |
| Service | Yes — translates vendor exceptions to domain exceptions, never swallows |
| Query | No — let propagate |
| Action | Rare — only with explicit business fallback (`// @reason ...`) |
| Listener / Job `handle()` | No — queue's retry mechanism handles failure |
| Listener / Job `failed()` | Yes — log/notify after retries exhausted |
| Controller | No — let propagate to handler |
| FormRequest | No — `authorize()` returns bool |
| Global exception handler (`bootstrap/app.php`) | Yes — catches everything, translates to HTTP, reports |

**Never catch `\Exception` or `\Throwable` in business code.**

### Central handler example

```php
// bootstrap/app.php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->render(function (InsufficientInventoryException $e, Request $request) {
        return response()->json([
            'message' => $e->getMessage(),
            'unavailable_items' => $e->unavailableItems,
        ], 422);
    });

    $exceptions->render(function (PaymentGatewayException $e) {
        report($e);
        return response()->json(['message' => 'Payment processing failed.'], 502);
    });

    $exceptions->dontReport([
        InsufficientInventoryException::class,
        PaymentDeclinedException::class,
    ]);
})
```

---

## Model — data shape, relationships, plumbing

**File:** `app/Models/{Model}/{Model}.php`
**Lifecycle hooks:** `boot()` / `booted()` directly in the model (or via traits in `app/Concerns/` for cross-cutting concerns)

### Skeleton

```php
<?php

declare(strict_types=1);

namespace App\Models\Order;

use App\Concerns\HasUuid;
use App\Models\Order\States\OrderState;
use App\Models\User;
use App\Services\OrderNumberGeneratorService;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Spatie\ModelStates\HasStates;

final class Order extends Model
{
    use HasFactory, HasUuid, HasStates;

    protected $fillable = [
        'number', 'customer_name', 'phone', 'notes',
        'total_amount', 'status', 'tracking_number',
        'shipped_at', 'merchant_id',
    ];

    protected $casts = [
        'status' => OrderState::class,
        'total_amount' => 'integer',
        'shipped_at' => 'datetime',
    ];

    protected static function booted(): void
    {
        static::creating(function (Order $order) {
            $order->number ??= app(OrderNumberGeneratorService::class)->next();
        });

        static::deleted(function (Order $order) {
            $order->items()->delete();
        });
    }

    public function merchant(): BelongsTo
    {
        return $this->belongsTo(User::class, 'merchant_id');
    }

    public function items(): HasMany
    {
        return $this->hasMany(OrderItem::class);
    }
}
```

### Model rules

1. Models contain:
   - Table config (`$table`, `$primaryKey`, `$keyType`, `$incrementing`)
   - `$fillable` (always explicit, NEVER `$guarded = []`)
   - `$casts`
   - Relationships
   - Accessors / mutators
   - Scopes
   - Trait usage
   - `boot()` / `booted()` lifecycle hooks
2. Models do NOT contain:
   - Business logic (decisions)
   - External API calls
   - Side effects beyond the model itself (no SMS, email, webhooks, event firing)
3. `final` by default
4. Cross-cutting concerns (UUIDs across many models, audit logging) go in `app/Concerns/` traits using `boot{Name}()` convention
5. Observer classes are NOT used — `boot()`/`booted()` in the model OR traits in `Concerns/`

### Trait skeleton

```php
<?php

declare(strict_types=1);

namespace App\Concerns;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;

trait HasUuid
{
    protected static function bootHasUuid(): void
    {
        static::creating(function (Model $model) {
            $model->uuid ??= (string) Str::uuid();
        });
    }
}
```

---

## State machines (when 3+ states)

**Package:** `spatie/laravel-model-states`
**File location:** `app/Models/{Model}/States/`

### Skeleton — abstract base

```php
<?php

declare(strict_types=1);

namespace App\Models\Order\States;

use Spatie\ModelStates\State;
use Spatie\ModelStates\StateConfig;

abstract class OrderState extends State
{
    abstract public function label(): string;

    public static function config(): StateConfig
    {
        return parent::config()
            ->default(Pending::class)
            ->allowTransition(Pending::class, Shipped::class, Transitions\PendingToShipped::class)
            ->allowTransition(Shipped::class, Delivered::class)
            ->allowTransition(Pending::class, Cancelled::class);
    }
}
```

### Concrete state

```php
<?php

declare(strict_types=1);

namespace App\Models\Order\States;

final class Pending extends OrderState
{
    public function label(): string
    {
        return 'Pending';
    }
}
```

### Transition (for non-trivial state changes)

```php
<?php

declare(strict_types=1);

namespace App\Models\Order\States\Transitions;

use App\Models\Order\Order;
use App\Models\Order\States\Shipped;
use Spatie\ModelStates\Transition;

final class PendingToShipped extends Transition
{
    public function __construct(
        private Order $order,
        private string $trackingNumber,
    ) {}

    public function handle(): Order
    {
        $this->order->status = new Shipped($this->order);
        $this->order->tracking_number = $this->trackingNumber;
        $this->order->shipped_at = now();
        $this->order->save();

        return $this->order;
    }
}
```

### State machine rules

1. Use state machines for any model with 3+ states or transition rules
2. Plain booleans (`is_active`, `is_published`) don't need state machines
3. State classes co-located with model: `app/Models/{Model}/States/`
4. Transition classes only when transition has non-trivial logic (sets extra fields, has validation)
5. Trivial transitions: call `$model->status->transitionTo(NewState::class)` from the Action directly — no Transition class
6. Transition does atomic state change. Events fire in the Action, not the Transition

---

## Sync vs async — the decision rule

**Question:** Does the HTTP response need the result of this work?

- **YES** → work runs **synchronously inside the Action**, inside `DB::transaction()` if it modifies the database
- **NO** → work runs **asynchronously after the Action**, via Event + queued Listener, or directly dispatched Job

### Examples

| Operation | Sync or async? |
|---|---|
| Calculating loyalty points awarded by an order (response shows "you earned 150 points") | **Sync** — inside Action |
| Sending order confirmation SMS | **Async** — queued Listener |
| Generating invoice PDF for immediate download | **Sync** — inside Action |
| Generating invoice PDF emailed later | **Async** — Job |
| Decrementing inventory (order invalid if not decremented) | **Sync** — inside Action's transaction |
| Notifying merchant of new order | **Async** — queued Listener |

### Actions dispatch Jobs directly ONLY for:

- Delayed work (`->delay()`)
- Scheduled work (from console scheduler)
- Chained jobs with strict ordering (`Bus::chain([...])`)
- Batched jobs (`Bus::batch([...])`)

For everything else, fire an Event and let Listeners react.

---

## Per-field-group updates

When an entity has distinct update flows with different validation/authorization/side effects, use separate FormRequest + Dto + Action per flow. Example for User:

```
app/Http/Requests/Users/UpdateUserNameRequest.php
app/Http/Requests/Users/UpdateUserEmailRequest.php
app/Http/Requests/Users/UpdateUserAddressRequest.php

app/Data/Users/UpdateUserNameDto.php
app/Data/Users/UpdateUserEmailDto.php
app/Data/Users/UpdateUserAddressDto.php

app/Actions/Users/UpdateUserNameAction.php
app/Actions/Users/UpdateUserEmailAction.php      # may trigger re-verification
app/Actions/Users/UpdateUserAddressAction.php

app/Http/Resources/UserResource.php              # single output Resource

Routes:
  PATCH /profile/name     → ProfileController@updateName
  PATCH /profile/email    → ProfileController@updateEmail
  PATCH /profile/address  → ProfileController@updateAddress
```

Use a single combined `UpdateUserDto` with optional fields ONLY when one form legitimately edits all fields together (admin "edit user" page).

---

## Decision tree — where does this code go?

```
Is it a write operation triggered by an HTTP request?
  → Controller method → FormRequest (validate) → Dto::from() → Action::handle()

Is it a read operation, with 3+ filters or aggregation or reuse across endpoints?
  → Query class with handle() method

Is it a simple read (Model::find($id))?
  → Inline in controller, no Query class needed

Is it a wrapper around an external API or library or stateful capability?
  → Service class (with interface if mocked in tests)

Is it a discrete reaction to an event (one job each)?
  → Listener implementing ShouldQueue

Is it async work — bulk import, scheduled task, webhook handler, retry-required external call?
  → Job implementing ShouldQueue

Is it a model-level invariant (UUID, slug, audit log, cascade cleanup)?
  → boot()/booted() in the model, OR a trait in app/Concerns/

Is it a discrete state change in a model with 3+ states?
  → State + Transition class under app/Models/{Model}/States/

Is it formatting a model into JSON for HTTP output?
  → JsonResource

Is it carrying validated data between layers?
  → Dto

Is it a domain failure (business rule violated)?
  → Throw a domain exception from app/Exceptions/
```

---

## Forbidden patterns

When you see these in code, refactor immediately:

1. **Business logic in a controller method** → extract to Action
2. **Query building in a controller** → extract to Query
3. **`$request->validate([...])` inline in controller** → move to FormRequest
4. **Validation attributes on Dto properties** → move to FormRequest
5. **Resource accessing `Model::find()` or any query** → move logic to Action, pass result to Resource
6. **`app()` or `resolve()` inside Action/Query/Service/Job/Listener/Controller method bodies** → inject in constructor
7. **Non-final concrete class** → add `final`, or `abstract` if a base
8. **Action implementing `ShouldQueue`** → it's a Job, not an Action
9. **Listener doing multiple side effects in one `handle()`** → split into multiple Listeners
10. **Event fired inside a `DB::transaction()`** → fire after transaction commits
11. **Catching `\Exception` or `\Throwable` in business code** → catch a specific type or let propagate
12. **Class without the type suffix** (e.g., `PlaceOrder` instead of `PlaceOrderAction`) → rename
13. **`$guarded = []` on a model** → switch to explicit `$fillable`
14. **Business logic in `boot()`/`booted()` or a trait** → move to an Action
15. **Mixed flat + nested folder structure** for the same type → promote all to subfolders
16. **Observer class** → use `boot()`/`booted()` in the model or a trait in `Concerns/`
17. **Listener with subfolder** → listeners are always flat
18. **`Data` suffix on a DTO instead of `Dto`** → rename

---

## Quick reference — generation checklist

When asked to create a new endpoint, generate this exact set of files:

| File | Required? |
|---|---|
| `app/Http/Requests/{Domain}/{Verb}{Object}Request.php` | Yes (write endpoints) |
| `app/Data/{Domain}/{Verb}{Object}Dto.php` | Yes (write endpoints) |
| `app/Actions/{Domain}/{Verb}{Object}Action.php` | Yes (write endpoints) |
| `app/Http/Resources/{Model}Resource.php` | Yes if not already existing |
| `app/Http/Controllers/{Resource}Controller.php` | Yes (or invokable controller) |
| Route definition in `routes/api.php` or `routes/web.php` | Yes |
| Event class if Action fires one | Per-Action |
| Listener classes for the Event | Per reaction |
| `tests/Feature/{Domain}/{Verb}{Object}Test.php` | Yes for critical paths |
| `tests/Unit/Actions/{Domain}/{Verb}{Object}ActionTest.php` | Yes for critical paths |

Always show the complete file with namespace, `<?php`, `declare(strict_types=1);`, and proper `use` statements. Never produce partial snippets unless explicitly asked.
