# DTO (Data Transfer Object)

## Definition

A typed, immutable object that **carries validated data between layers**. DTOs do nothing else: no validation, no business logic, no serialization formatting. They are pure transport.

Built on `spatie/laravel-data` for the convenience of `::from()` hydration (auto-construct from arrays, models, requests). The validation features of `spatie/laravel-data` are NOT used — validation lives in FormRequests.

## Keywords

Data Transfer Object • DTO • Payload • Typed structure • Transport object

## Purpose

To replace ad-hoc arrays between layers with explicit, typed contracts. A DTO is a single declaration of "this is the shape of data flowing here." Type checkers and IDEs catch mistakes. Refactoring is safe.

## Rules

1. **One DTO per shape.** Input DTOs and filter DTOs are separate.
2. **No validation.** No `#[Required]`, `#[Email]`, etc. attributes. Validation lives in FormRequests.
3. **All properties `public readonly`.** Once constructed, DTOs are immutable.
4. **All DTOs `final`.**
5. **No methods beyond the constructor** (and inherited `::from()`). No calculations, no formatting, no business logic.
6. **No TypeScript attributes.** Frontend types maintained separately.
7. Suffix is **`Dto`**, not `Data`.

### Naming

| Pattern | Use case | Example |
|---|---|---|
| `Create{X}Dto` | Input for POST | `CreateOrderDto` |
| `Update{X}Dto` | Input for PUT/PATCH (all fields together) | `UpdateOrderDto` |
| `Update{X}{Field}Dto` | Input for partial / scoped update | `UpdateUserNameDto`, `UpdateUserEmailDto` |
| `{X}FiltersDto` | Query parameters for read endpoints | `OrderFiltersDto` |
| `{X}Dto` | Generic inter-layer transport | `OrderConfirmationDto` |

## Shape

```php
namespace App\Data\{Domain};

use Spatie\LaravelData\Data;

final class {Name}Dto extends Data
{
    public function __construct(
        public readonly string $field,
        public readonly ?int $nullable_field,
    ) {}
}
```

## Parameters

Constructor properties. Each property is a typed, readonly field.

## Returns

N/A — DTOs are themselves the result of `::from()`.

## Is it `final`?

**Yes. Always.**

## When to use

- **Carrying validated FormRequest data** into an Action
- **Passing structured arguments** between Actions, Services, Queries
- **Defining filter parameters** for Queries
- **Defining payloads** for Events and Jobs
- **Inter-layer transport** when an array would lose type information

## When NOT to use

- **For input validation** → that's FormRequest's job
- **For output serialization** → that's JsonResource's job
- **For wrapping a single primitive** → just use the primitive
- **As a class with logic methods** → DTOs are dumb data holders

## Lifecycle

1. FormRequest validates incoming HTTP data
2. Controller calls `{X}Dto::from($request->validated())`
3. `spatie/laravel-data` constructs the DTO with typed, readonly properties
4. DTO is passed to an Action as a parameter
5. Action accesses DTO properties directly: `$dto->customer_name`
6. DTO is discarded when the request ends

## What controls it

- **Constructed by:** Controllers (from validated request data), Actions (when building intermediate DTOs), Jobs/Listeners (from passed parameters)
- **Hydrated via:** `::from(array|object|Model|other)` — handles arrays, models, other DTOs
- **No registration** — DTOs are not container-bound; they're plain value objects

## Examples

### Basic input DTO

```php
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

### Nested DTO

```php
final class OrderItemDto extends Data
{
    public function __construct(
        public readonly int $product_id,
        public readonly int $quantity,
    ) {}
}
```

### Filter DTO

```php
use Carbon\Carbon;

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

### Per-field-group update DTOs

```php
// app/Data/Users/
final class UpdateUserNameDto extends Data
{
    public function __construct(
        public readonly string $name,
    ) {}
}

final class UpdateUserEmailDto extends Data
{
    public function __construct(
        public readonly string $email,
    ) {}
}

final class UpdateUserAddressDto extends Data
{
    public function __construct(
        public readonly string $street,
        public readonly string $city,
        public readonly string $postal_code,
        public readonly string $country,
    ) {}
}
```

### Composite return DTO (Action returns multiple values)

When an Action needs to return more than a single Model — e.g., the order plus computed loyalty points — use a return DTO:

```php
namespace App\Data\Orders;

use App\Models\Order\Order;
use Spatie\LaravelData\Data;

final class OrderConfirmationDto extends Data
{
    public function __construct(
        public readonly Order $order,
        public readonly int $points_earned,
    ) {}
}
```

```php
// In the Action
public function handle(CreateOrderDto $dto): OrderConfirmationDto
{
    return DB::transaction(function () use ($dto) {
        $order = Order::create([...]);
        $points = $this->loyalty->awardPointsFor($order);

        return new OrderConfirmationDto(
            order: $order,
            points_earned: $points,
        );
    });
}
```

### Usage in a controller

```php
public function store(StoreOrderRequest $request, PlaceOrder $action)
{
    $dto = CreateOrderDto::from($request->validated());
    $order = $action->handle($dto);
    return new OrderResource($order);
}
```

The flow:
1. `StoreOrderRequest` validates the incoming request body
2. `$request->validated()` returns a clean array of validated fields
3. `CreateOrderDto::from(...)` constructs the typed DTO from that array
4. The Action receives a typed object, not a loose array

### Constructing DTOs manually (outside HTTP context)

DTOs are not coupled to requests — they work from anywhere:

```php
// In a console command
$dto = new CreateOrderDto(
    customer_name: 'Test Customer',
    phone: '0912345678',
    items: [new OrderItemDto(product_id: 1, quantity: 2)],
);
$action->handle($dto);

// Or from another source
$dto = CreateOrderDto::from([
    'customer_name' => 'Test Customer',
    'phone' => '0912345678',
    'items' => [['product_id' => 1, 'quantity' => 2]],
]);
```
